---
id: 1466
title: buildZodSchema root input schemas strip unknown top-level keys instead of failing fast
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - security
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-04T13:50:25Z'
updatedAt: '2026-11-04T19:15:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1466'
author: neo-gemini-julia
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-04T19:15:51Z'
---
# buildZodSchema root input schemas strip unknown top-level keys instead of failing fast

## Context

While posting a comment intended for a non-default repository (`<org>/<repo>#15`) during session `50cec9b3-cbdc-4b13-befe-10aaa76a14d1` (2026-11-04 13:44Z), calling `manage_issue_comment` with `repository: '<org>/<repo>'` (instead of `repo: '<org>/<repo>'`) succeeded with zero validation error: the unknown top-level key `repository` was silently stripped by Zod (`z.object(shape)`), `repo` defaulted to `undefined`, and `CommentService` posted the comment onto `neomjs/neo#15` (immediately deleted at 13:45Z via `DELETE /repos/neomjs/neo/issues/comments/6632232801`).

## The Problem

Every MCP tool's advertised JSON Schema (`toOpenApiJsonSchema(buildZodSchema(doc, op))`) emits `"additionalProperties": false` at the root object because Zod v4's `z.toJSONSchema(z.object(shape))` emits `"additionalProperties": false` for both strip-mode (`z.object(shape)`) and strict-mode (`z.object(shape).strict()`) schemas.

Because of this Zod v4 JSON Schema behavior, the compliance test added for `#9837` (`test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs:1222-1247`, *"every emitted input schema stays strict at the root (regression guard against #9837 overreach)"*) asserts:
```js
// we only guard the top-level contract surface here so agents passing unknown top-level fields fail fast.
if (inputSchema.type === 'object') {
    checkedOps++;
    if (inputSchema.additionalProperties === false) strictOps++;
}
```
That assertion passes (`strictOps === checkedOps`) even when `buildZodSchema` returns a non-strict `z.object(shape)` that **silently strips** unknown top-level keys at runtime instead of rejecting them!

In `ai/mcp/validation/openApiValidator.mjs:66-108`, `buildZodSchema` previously initialized `let strictRoot = false` and only set `strictRoot = requestBodySchema.additionalProperties === false`. For any operation whose `openapi.yaml` uses query parameters (`operation.parameters`) or omits an explicit `additionalProperties: false` on `requestBody.content['application/json'].schema` (including all 25 `operationId` operations in `ai/mcp/server/github-workflow/openapi.yaml`), `buildZodSchema` returned a strip-mode `z.object(shape)`. Consequently, passing a misspelled or alias top-level argument (`repository` vs `repo`, `prNumber` vs `pr_number`, etc.) silently dropped the caller's argument and executed against default values (such as the default public repository `neomjs/neo`).

At the same time, several existing operations in `knowledge-base` and `memory-core` intentionally omit `additionalProperties: false` and rely on strip-at-boundary behavior when internal or tenant-authority fields (`tenantId`, `materializationAttempt`, `staleStrategy`, `viaMcp`, `custodian`, `depth`) are withheld from the public OpenAPI surface (`KBCrossTenantIsolation.integration.spec.mjs`, `GoldenPathSynthesizer.mjs`).

## The Architectural Reality

- `ai/mcp/validation/openApiValidator.mjs#buildZodSchema` (`lines 64-126`): builds the top-level input `z.object(shape)` from `operation.parameters` and `operation.requestBody.content['application/json'].schema`.
- `ai/mcp/server/github-workflow/openapi.yaml`: defines 25 `operationId` operations (spanning both `operation.parameters` GET tools and `requestBody` POST tools) where unknown top-level argument keys must fail fast at runtime rather than silently falling back to default repository/target parameters.
- `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs`: guards OpenAPI-to-Zod schema compilation across all 5 MCP servers.
- **Design authority:** `#9837` and `OpenApiValidatorCompliance.spec.mjs`, plus `AGENTS.md §critical_gates` #9 (preventing silent fallback from a private target repo to the default public repo when `repository` is passed instead of `repo`).

## Contract Ledger (`x-neo-strict-input`)

- **Extension key & placement:** `x-neo-strict-input: true` (boolean OpenAPI vendor extension accepted at document root `openApiDocument['x-neo-strict-input']` or operation root `operation['x-neo-strict-input']`).
- **Compiler consumer (`ai/mcp/validation/openApiValidator.mjs#buildZodSchema`):**
  - Initializes `let strictRoot = Boolean(openApiDocument?.['x-neo-strict-input'] === true || operation?.['x-neo-strict-input'] === true)`.
  - **Precedence rules:**
    1. `requestBodySchema.additionalProperties === false` sets `strictRoot = true` (explicit operation-level schema strictness).
    2. Truthy `requestBodySchema.additionalProperties` (`true` or schema object) sets `strictRoot = false` (explicitly open request body overrides document/operation `x-neo-strict-input: true`).
    3. Otherwise, `strictRoot` retains the document/operation `x-neo-strict-input === true` opt-in, applying `.strict()` to the compiled root `z.object(shape)` for both `operation.parameters` (GET) and `requestBody` (POST/PUT/PATCH) operations.
- **Active producer:** `ai/mcp/server/github-workflow/openapi.yaml` (`x-neo-strict-input: true` at document root; 25/25 `operationId` operations reject unknown top-level keys with `unrecognized_keys`).
- **Preserved strip-at-boundary consumers:** Servers/operations omitting both `x-neo-strict-input: true` and `requestBodySchema.additionalProperties: false` retain strip-mode `z.object(shape)` at the root (`knowledge-base` 0/13 strict, preserving `KBCrossTenantIsolation.integration.spec.mjs` forged-`tenantId` stripping on `query_documents` / `ask_knowledge_base`; `memory-core` 7/55 explicit `additionalProperties: false` strict, preserving `GoldenPathSynthesizer.mjs` `getContextFrontier({ depth: 1 })` and withheld authority-field stripping on the remaining operations; `file-system` 0/7; `neural-link` 1/60).

## The Fix

1. In `ai/mcp/validation/openApiValidator.mjs#buildZodSchema`:
   - Honor `x-neo-strict-input: true` at the OpenAPI document or operation level alongside `requestBodySchema.additionalProperties === false` (unless `requestBodySchema.additionalProperties` explicitly permits additional properties).
2. In `ai/mcp/server/github-workflow/openapi.yaml`:
   - Enable `x-neo-strict-input: true` at the document root so all 25 `github-workflow` operations reject unknown top-level keys at runtime.
3. In `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs`:
   - Extend the root-strictness compliance test so any operation opting in via `x-neo-strict-input: true` (at document or operation scope) is verified at runtime via `zodSchema.safeParse({ __unexpected_top_level_key__: true })` failing with `unrecognized_keys`.
   - Add a direct regression test for `github-workflow` verifying that all 25 `operationId` entries reject `repository` (instead of `repo`) with `unrecognized_keys`.

## Acceptance Criteria

- [x] **AC-1 (`buildZodSchema` `x-neo-strict-input` opt-in + `github-workflow` producer):** `ai/mcp/validation/openApiValidator.mjs#buildZodSchema` honors `x-neo-strict-input: true` at the OpenAPI document or operation level (alongside `requestBody.content['application/json'].schema.additionalProperties === false`, unless `additionalProperties` explicitly permits extra properties), and `ai/mcp/server/github-workflow/openapi.yaml` declares `x-neo-strict-input: true` so all 25 `github-workflow` operations reject unknown top-level keys at runtime while preserving strip-at-boundary compatibility for `knowledge-base`, `memory-core`, `file-system`, and `neural-link`.
- [x] **AC-2 (`OpenApiValidatorCompliance.spec.mjs` runtime verification):** The per-server root-strictness compliance test in `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs` verifies runtime rejection of unknown top-level keys (`unrecognized_keys`) for all schemas opting in via `x-neo-strict-input: true`, plus an explicit `github-workflow` regression test confirming all 25 `operationId` entries reject `repository` (vs `repo`) with `unrecognized_keys`.
- [x] **AC-3 (Test suite & compatibility preservation):** `OpenApiValidatorCompliance.spec.mjs`, `GoldenPathSynthesizer.spec.mjs`, and `KBCrossTenantIsolation.integration.spec.mjs` pass with zero regressions.

## Out of Scope

- Changing `repo` parameter names in `ai/mcp/server/github-workflow/openapi.yaml` (keeping `repo` canonical while failing closed on `repository` prevents silent fallback without expanding schema parameter count).
- Modifying output schema leniency (`buildOutputZodSchema` remains `.passthrough()` per `#9837`).
- Global strict-root enforcement across `knowledge-base` or `memory-core` operations that rely on boundary stripping for withheld internal/tenant authority fields.

## Decision Record impact

`none` — introduces the opt-in `x-neo-strict-input` OpenAPI extension documented in the Contract Ledger above and enables it on `github-workflow`.

## Related

- `#9837`, `#10064`, `#10070`
- Promoted from `defect-note` `MESSAGE:2c579dd5-d221-4c09-ab63-c0bac46174fa` (`a2369af8ed68a0e6`).

Live latest-open sweep: checked latest 20 open issues on `neomjs/neo-agent-brain` (`sort: 'created'`, `#1438`..`#1010`) and searched `strictRoot` / `openApiValidator strict` at 2026-11-04T13:49Z; no equivalent found. A2A in-flight claim sweep (`list_messages({ status: 'all', limit: 30 })`): no competing claim. Memory Core rationale sweep (`openApiValidator buildZodSchema strictRoot additionalProperties unknown top-level fields 9837`): confirmed `#9837` intended top-level input schemas to fail fast on unknown fields. Own-assignment sweep: 1 open (`#1230`), non-overlapping. Structure map (`npm run ai:structure-map -- --files --loc`): `ai/mcp/validation/openApiValidator.mjs` owns OpenAPI-to-Zod compilation.

Origin Session ID: 50cec9b3-cbdc-4b13-befe-10aaa76a14d1
Retrieval Hint: "openApiValidator buildZodSchema strictRoot x-neo-strict-input additionalProperties unrecognized_keys repository repo github-workflow"


## Timeline

- 2026-11-04T13:50:25Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-04T13:50:26Z @neo-gemini-julia added the `bug` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `ai` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `model-experience` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `security` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `agent-os` label
- 2026-11-04T14:03:52Z @neo-gemini-julia cross-referenced by PR #1467
- 2026-11-04T14:17:43Z @neo-gemini-julia referenced in commit `9ffafa8` - "fix(mcp): scope root strictness via x-neo-strict-input on github-workflow (#1466)"
- 2026-11-04T19:15:51Z @tobiu referenced in commit `f1fc04b` - "fix(mcp): enforce runtime root strictness in buildZodSchema (#1466) (#1467)

* fix(mcp): enforce runtime root strictness in buildZodSchema (#1466)

* fix(mcp): scope root strictness via x-neo-strict-input on github-workflow (#1466)"
- 2026-11-04T19:15:53Z @tobiu closed this issue

