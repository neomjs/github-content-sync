---
id: 1466
title: buildZodSchema root input schemas strip unknown top-level keys instead of failing fast
state: OPEN
labels:
  - bug
  - ai
  - model-experience
  - security
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-04T13:50:25Z'
updatedAt: '2026-11-04T14:18:09Z'
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

In `ai/mcp/validation/openApiValidator.mjs:66-108`, `buildZodSchema` initializes `let strictRoot = false` and only sets `strictRoot = requestBodySchema.additionalProperties === false`. For any operation whose `openapi.yaml` uses query parameters (`operation.parameters`) or omits an explicit `additionalProperties: false` on `requestBody.content['application/json'].schema` (including all 22 operations in `ai/mcp/server/github-workflow/openapi.yaml`), `buildZodSchema` returns a strip-mode `z.object(shape)`. Consequently, passing a misspelled or alias top-level argument (`repository` vs `repo`, `prNumber` vs `pr_number`, etc.) silently drops the caller's argument and executes against default values (such as the default public repository `neomjs/neo`).

## The Architectural Reality

- `ai/mcp/validation/openApiValidator.mjs#buildZodSchema` (`lines 64-109`): builds the top-level input `z.object(shape)` from `operation.parameters` and `operation.requestBody.content['application/json'].schema`.
- `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs:1222-1247`: documents the `#9837` contract that top-level MCP tool input schemas must reject unknown top-level fields fast (`additionalProperties: false` at runtime as well as in `tools/list`), while output schemas (`buildOutputZodSchema`, `lenient: true`) use `.passthrough()`.
- **Design authority:** `#9837` and `OpenApiValidatorCompliance.spec.mjs:1222-1247` (*"we only guard the top-level contract surface here so agents passing unknown top-level fields fail fast"*), plus `AGENTS.md §critical_gates` #9 (preventing silent fallback from a private target repo to the default public repo when `repository` is passed instead of `repo`).

## The Fix

1. In `ai/mcp/validation/openApiValidator.mjs#buildZodSchema`:
   - Default `let strictRoot = true` for the top-level tool input `z.object(shape)`, only disabling `strictRoot` when `requestBodySchema.additionalProperties` explicitly permits additional properties (`true` or a schema object).
2. In `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs`:
   - Strengthen the `#9837` root-strictness compliance test across all 5 MCP servers so that in addition to checking `inputSchema.additionalProperties === false` in the emitted JSON Schema, it verifies that `zodSchema.safeParse({ __unexpectedTopLevelKey__: true })` fails with `unrecognized_keys` (proving runtime `.strict()` enforcement, not merely JSON Schema emission).
   - Add a direct regression test for `github-workflow` operations (`manage_issue_comment`, `get_conversation`, `manage_pr_review`, `create_issue`) confirming that passing `repository` instead of `repo` is rejected by `buildZodSchema` with `unrecognized_keys`.

## Acceptance Criteria

- [ ] **AC-1 (`buildZodSchema` runtime root strictness):** `ai/mcp/validation/openApiValidator.mjs#buildZodSchema` returns a `.strict()` top-level `z.object(shape)` by default unless `requestBody.content['application/json'].schema.additionalProperties` explicitly allows extra properties (`true` or object schema).
- [ ] **AC-2 (`OpenApiValidatorCompliance.spec.mjs` runtime verification):** The per-server root-strictness compliance test in `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs` verifies runtime rejection of unknown top-level keys (`unrecognized_keys`) across all 5 MCP servers (`file-system`, `github-workflow`, `knowledge-base`, `memory-core`, `neural-link`), plus an explicit `github-workflow` `repository` vs `repo` regression test.
- [ ] **AC-3 (Unit test suite green):** Any existing unit test callers passing undeclared top-level keys to `buildZodSchema` are aligned with their `openapi.yaml` contracts, and `OpenApiValidatorCompliance.spec.mjs` passes.

## Out of Scope

- Changing `repo` parameter names in `ai/mcp/server/github-workflow/openapi.yaml` (keeping `repo` canonical while failing closed on `repository` prevents silent fallback without expanding schema parameter count).
- Modifying output schema leniency (`buildOutputZodSchema` remains `.passthrough()` per `#9837`).

## Decision Record impact

`none` — enforces the existing `#9837` top-level input strictness contract at runtime.

## Related

- `#9837`, `#10064`, `#10070`
- Promoted from `defect-note` `MESSAGE:2c579dd5-d221-4c09-ab63-c0bac46174fa` (`a2369af8ed68a0e6`).

Live latest-open sweep: checked latest 20 open issues on `neomjs/neo-agent-brain` (`sort: 'created'`, `#1438`..`#1010`) and searched `strictRoot` / `openApiValidator strict` at 2026-11-04T13:49Z; no equivalent found. A2A in-flight claim sweep (`list_messages({ status: 'all', limit: 30 })`): no competing claim. Memory Core rationale sweep (`openApiValidator buildZodSchema strictRoot additionalProperties unknown top-level fields 9837`): confirmed `#9837` intended top-level input schemas to fail fast on unknown fields. Own-assignment sweep: 1 open (`#1230`), non-overlapping. Structure map (`npm run ai:structure-map -- --files --loc`): `ai/mcp/validation/openApiValidator.mjs` owns OpenAPI-to-Zod compilation.

Origin Session ID: 50cec9b3-cbdc-4b13-befe-10aaa76a14d1
Retrieval Hint: "openApiValidator buildZodSchema strictRoot additionalProperties unrecognized_keys repository repo github-workflow"


## Timeline

- 2026-11-04T13:50:25Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-04T13:50:26Z @neo-gemini-julia added the `bug` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `ai` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `model-experience` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `security` label
- 2026-11-04T13:50:27Z @neo-gemini-julia added the `agent-os` label
- 2026-11-04T14:03:52Z @neo-gemini-julia cross-referenced by PR #1467
- 2026-11-04T14:17:43Z @neo-gemini-julia referenced in commit `9ffafa8` - "fix(mcp): scope root strictness via x-neo-strict-input on github-workflow (#1466)"

