---
id: 1886
title: 'Omit empty properties:{} on additionalProperties map schemas in openApiValidator.mjs'
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - model-experience
  - agent-os
assignees:
  - utsxvrai
createdAt: '2026-11-21T07:26:43Z'
updatedAt: '2026-11-21T08:58:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1886'
author: neo-gemini-julia
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# Omit empty properties:{} on additionalProperties map schemas in openApiValidator.mjs

Please comment to claim before starting; check current assignees and linked PRs.

## Context

While submitting a same-head Round-2 `APPROVED` review on `neomjs/neo#21698` via `call_mcp_tool` (`neo-mjs-github-workflow` → `manage_pr_review`) in Antigravity, passing `acknowledgedRequestChanges: {"neo-gemini-julia": "superior-evidence: ..."}` failed repeatedly with:

```text
Missing acknowledgment(s): @neo-gemini-julia.
```

even though the key was explicitly supplied in `Arguments.acknowledgedRequestChanges`.

## The Problem

In `ai/mcp/validation/openApiValidator.mjs`, `buildZodSchemaFromNode()` (`L186–L221`) builds map/open-bag object schemas as `z.object({}).catchall(additionalSchema)` or `z.object({}).passthrough()`. When `toOpenApiJsonSchema()` (`L15–L54`) serializes those Zod v4 schemas via `z.toJSONSchema(zodSchema, {target: 'openapi-3.0'})`, Zod v4 emits an empty `"properties": {}` object alongside `"additionalProperties"` — for example in `manage_pr_review`:

```json
"acknowledgedRequestChanges": {
  "additionalProperties": {"type": "string"},
  "properties": {},
  "type": "object"
}
```

Strict MCP tool-invocation bridges (confirmed live in Antigravity's `call_mcp_tool` across **27** Neo MCP tool schemas under `<appDataDir>/mcp/neo-mjs-*/*.json`, including `github-workflow` `manage_pr_review`, `knowledge-base` `ingest_source_files`, `memory-core` `admit_community_batch` / `save_nl_transaction`, and `neural-link` `set_instance_properties` / `find_instances` / `query_component` / `modify_state_provider` / `call_method` / `create_component` / `create_instance` / `simulate_event`) treat a non-absent `"properties": {}` map as the closed allowlist of keys and silently strip dynamic map keys before sending the JSON-RPC `tools/call` request, delivering `{}` to `ToolService#callTool`.

## The Architectural Reality

- `ai/mcp/validation/openApiValidator.mjs` `normalizeOpenApiJsonSchema(node)` (`L26–L54`) already normalizes Zod v4's `z.toJSONSchema()` output to match strict MCP client contracts (e.g. normalizing `additionalProperties: {}` → `additionalProperties: true` at `L40–L47` and nullable enums at `L31–L33`), while `ToolService#callTool` (`ai/mcp/ToolService.mjs:139`) still validates runtime arguments through the server-side `z.object({}).catchall(...)` / `.passthrough()` Zod instance.
- In JSON Schema / OpenAPI 3.0, omitting `"properties"` on a pure map/record (`type: 'object'` with zero declared fixed properties and `additionalProperties !== false`) is semantically identical to `"properties": {}`, while avoiding the empty-struct branch in strict client-side schema marshalers.
- Structure map (`npm run ai:structure-map -- --files --loc`): `ai/mcp/validation/openApiValidator.mjs` is the single canonical owner of `tools/list` JSON Schema normalization across all Neo MCP servers.
- Design authority: `ai/mcp/validation/openApiValidator.mjs:4-19` & `L212-220` (*"Without passthrough, Zod's strict parse silently strips the entire payload to `{}`"* — the emitted `tools/list` JSON Schema must preserve the same open-key contract on the client side).

## The Fix

1. In `normalizeOpenApiJsonSchema(node)` (`ai/mcp/validation/openApiValidator.mjs`), when `node.type === 'object'`, `node.additionalProperties !== false && node.additionalProperties !== undefined`, and `node.properties` is a plain object with `Object.keys(node.properties).length === 0`, `delete node.properties` (preserving non-empty `node.properties` on hybrid fixed+catchall objects and preserving `properties: {}` when `additionalProperties === false`).
2. Add a unit test in `test/playwright/unit/ai/mcp/validation/` verifying that `toOpenApiJsonSchema()` omits empty `properties: {}` on both `additionalProperties: true` (open-bag) and typed `additionalProperties: {type: 'string'}` (`acknowledgedRequestChanges`-style record maps), while retaining `properties` when declared properties exist or `additionalProperties === false`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `toOpenApiJsonSchema` / `normalizeOpenApiJsonSchema` (`ai/mcp/validation/openApiValidator.mjs`) | `ai/mcp/validation/openApiValidator.mjs:15-54` | Omits empty `properties: {}` on `type: 'object'` nodes that allow `additionalProperties` (`true` or subschema), keeping runtime Zod `.catchall()` / `.passthrough()` parsing unchanged | Retains `properties` whenever `Object.keys(properties).length > 0` or `additionalProperties === false` | JSDoc on `normalizeOpenApiJsonSchema` | Playwright unit test in `test/playwright/unit/ai/mcp/validation/` |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `toOpenApiJsonSchema()` omits `"properties": {}` on `type: "object"` schemas with zero fixed properties and `additionalProperties: true` or `additionalProperties: <schema>` (including `manage_pr_review`'s `acknowledgedRequestChanges` and `neural-link`'s open-bag inputs).
- [ ] `toOpenApiJsonSchema()` preserves `properties` when at least one fixed property is declared or when `additionalProperties === false`.
- [ ] Unit tests under `test/playwright/unit/ai/mcp/validation/` cover open-bag (`additionalProperties: true`), typed map (`additionalProperties: {type: 'string'}`), hybrid fixed+additional object, and closed empty object (`additionalProperties: false`).

## Out of Scope

- Changing `ToolService#callTool` server-side Zod runtime validation.
- Modifying individual `openapi.yaml` tool contracts.

## Avoided Traps

- Deleting `properties: {}` when `additionalProperties === false` (an intentionally empty closed object must keep its closed shape).

## Related

- Surfaced during Round-2 review of `neomjs/neo#21698` (`manage_pr_review` `acknowledgedRequestChanges`).

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked latest 20 open (`160` open total, red band — standalone bug in `ai/mcp/validation/openApiValidator.mjs` with no open owner) + open and closed search (`additionalProperties openApiValidator acknowledgedRequestChanges`) at `2026-11-21T07:25Z`; no equivalent or prior exclusion found.
A2A in-flight sweep (`list_messages` `status: 'all'`, `limit: 30`), Memory Core rationale sweep (`acknowledgedRequestChanges properties additionalProperties call_mcp_tool stripped`), and own-assignment sweep (`#1230` only): zero overlap.

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181

Retrieval Hint: "openApiValidator normalizeOpenApiJsonSchema empty properties additionalProperties call_mcp_tool strips map keys"

## Timeline

- 2026-11-21T07:26:45Z @neo-gemini-julia added the `bug` label
- 2026-11-21T07:26:45Z @neo-gemini-julia added the `good first issue` label
- 2026-11-21T07:26:45Z @neo-gemini-julia added the `help wanted` label
- 2026-11-21T07:26:46Z @neo-gemini-julia added the `ai` label
- 2026-11-21T07:26:46Z @neo-gemini-julia added the `model-experience` label
- 2026-11-21T07:26:46Z @neo-gemini-julia added the `agent-os` label
### @utsxvrai - 2026-11-21T08:58:11Z

I'd like to claim this. Plan: strip the empty `properties: {}` that Zod v4's `toJSONSchema` emits next to `additionalProperties` for the `catchall` / `passthrough` map schemas in `toOpenApiJsonSchema()` (recursively, so nested maps are covered too), leaving objects with real properties untouched, plus tests for flat map, nested map and a normal object.

- 2026-11-21T08:58:35Z @github-actions assigned to @utsxvrai
### @github-actions - 2026-11-21T08:58:36Z

Thanks for claiming this, @utsxvrai, and welcome! Assigned to you — you can start now.

Your claim reserves this issue for 3 days (any comment here resets the clock; we still merge the first approve-worthy PR if another lands first or a claim goes quiet).

<sub>An automatic reply. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@utsxvrai -->

- 2026-11-21T09:11:50Z @utsxvrai cross-referenced by PR #1889

