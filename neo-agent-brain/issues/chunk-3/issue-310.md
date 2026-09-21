---
id: 310
title: An untyped response schema promises a result wrapper that object results never carry
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-04T13:55:32Z'
updatedAt: '2026-09-04T23:56:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/310'
author: neo-fable
commentsCount: 1
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
closedAt: '2026-09-04T23:56:06Z'
---
# An untyped response schema promises a result wrapper that object results never carry

## Context

Hit twice on 2026-09-04 — Vega's defect-note and my session `ee061205` during the D#18224 A1 arm: every `get_instance_properties` call through the MCP surface fails in the client with `data must have required property 'result'` (the harness validates `structuredContent` against the listed `outputSchema`). The same read succeeds through the raw WebSocket fixture (`neomjs/neo` `test/playwright/fixtures.mjs`), so specs never see it. Pinned at Brain `ea336dd` and engine `dev@2ee23d7801`.

## The Problem

Two code paths decide the response shape independently:

- `ai/mcp/validation/openApiValidator.mjs:293–296` (`buildOutputZodSchema`): when the 200 `application/json` schema resolves to anything but `type: object`, the listed output schema becomes `z.object({result: …})` — `result` required.
- `ai/mcp/server/BaseServer.mjs:526–545` (`formatToolResult`): an object result is emitted as `structuredContent` **unwrapped**; only non-objects are wrapped as `{result}`.

They agree for declared arrays and primitives and disagree for a declared-untyped response whose runtime value is an object. `get_instance_properties` (`ai/mcp/server/neural-link/openapi.yaml:246–277`) declares `schema: {description: 'The property values'}` with no `type`, and the engine returns a plain property map (`src/ai/client/InstanceService.mjs:26–37`, forwarded untouched by `ai/services/neural-link/InstanceService.mjs:190`). The listing promises `result`; the payload never carries it; every call fails validation.

Scan of all 59 operations (scratch script over `openapi.yaml`): 5 carry a non-object 200 schema and receive the wrapper — `get_instance_properties` (untyped), `get_console_logs` / `get_worker_topology` / `get_window_topology` (array), `simulate_event` (boolean); 17 declare no schema and list none. The untyped one is the only object-valued member, so this is the whole blast radius today — and the next untyped object-valued operation reproduces it.

## The Fix

1. **Contract:** declare what the engine returns — `{properties: <map>}` (`src/ai/client/InstanceService.mjs#getInstanceProperties` answers the requested values keyed by name under one `properties` key): `type: object`, `required: [properties]`, `properties.properties` an open object — so the builder emits that shape and the formatter's raw emission matches it. *(Truth-synced at review round 2: the ticket first described the payload as a bare map.)*
2. **Invariant:** `buildOutputZodSchema` wraps only a **declared non-object type** — exactly the set the formatter wraps. A schema with no top-level type is one of two things: a **composition** (`oneOf` / `anyOf` / `allOf`) keeps its declared branches — the root stays `type: object` (the MCP outputSchema contract) and the compiled branches ride beneath it as `anyOf` / `allOf` — while a **bare** schema lists as an open object. Neither promises `result`. *(Truth-synced at round 2: the first cut flattened compositions to an open object; `ingest_source_files` is a `oneOf` of two object refs and keeps both.)*
3. **Test:** a unit spec walks every operation of every OpenAPI document and asserts listing/formatter agreement from both sides: `required: ['result']` in a listed output schema if and only if the declared type is non-object, a composition carrying its branch count, and the real `BaseServer#formatToolResult` envelope validated against the emitted listing with the SDK's own Ajv provider (object result, declared primitive, the composed contract incl. a rejected malformed value, a bare probe). Red at `ea336dd` for the property reader and for the KB ingest composition.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `openapi.yaml` `/instance/properties/get` 200 schema | the tool contract + the engine's `getInstanceProperties` return | `type: object`, `required: [properties]`, `properties.properties` open | — | operation description (serialization note already there) | `get_instance_properties` succeeds through MCP — a real `tools/call` with client validation on |
| `buildOutputZodSchema` (`openApiValidator.mjs`, the typeless branch) | listing = formatter | composition ⇒ object root carrying its branches as `anyOf` / `allOf`; bare ⇒ open object; declared non-object ⇒ `{result}` | — | function docblock | the walk + envelope spec |

## Acceptance Criteria

- [ ] `get_instance_properties` on any live instance returns through the MCP surface with `structuredContent` validated (today: `data must have required property 'result'`) — a real `tools/call` with the SDK client's output validation active, not a listing-only probe.
- [ ] The listing/formatter agreement spec passes for every operation of all six OpenAPI documents and is red-first for the property reader (and for the KB ingest composition it turned up).
- [ ] No other operation's listed output schema changes except by declaring a type it already returns, or — for a composition with no top-level type — by moving its declared branches from beneath the `result` key the payload never carried to beneath the object root, fields and enums preserved.

## Out of Scope

- The 17 schema-less operations (unlisted output; a separate contract-completeness pass).
- `set_neo_config` dispatch — engine side, neomjs/neo#18281.

## Related

- neomjs/neo D#18224 (occurrences: Vega's defect-note; my A1 receipt DC_kwDODSospM4BFxN1) · neomjs/neo#18281 · `ai/mcp/ToolService.mjs:180–182, 227–239` (where the listed schema is attached).

Live latest-open sweep: the latest 25 open issues here (2026-09-04 13:52Z) — no equivalent. A2A claim sweep (last 60 min, all read-states): none on the Neural Link schema surface. Memory Core sweep: the two defect-notes above, no ticket. Own-assignment sweep: none in this repo.

unowned-rationale: a one-line contract fix plus one spec; claimable by any seat; found in peer-role, not on my active lanes.

Origin Session ID: ee061205-8271-4458-b73b-b132c1bec310

Retrieval Hint: `query_raw_memories("get_instance_properties required property result outputSchema wrapper formatToolResult untyped")`


## Timeline

- 2026-09-04T13:55:33Z @neo-fable added the `bug` label
- 2026-09-04T13:55:33Z @neo-fable added the `ai` label
- 2026-09-04T13:55:33Z @neo-fable added the `agent-os` label
### @neo-fable - 2026-09-04T13:57:14Z

Live positive control (2026-09-04 13:56Z, Agent OS app, session e2fb1159, instance `neo-button-2`, properties `[id, text]`): the MCP call fails in the client with `Structured content does not match the tool's output schema: data must have required property 'result'`. Same instance reads fine through `find_instances` with `returnProperties`, which declares an object response. — Mnemosyne, session ee061205-8271-4458-b73b-b132c1bec310

- 2026-09-04T22:59:57Z @neo-fable assigned to @neo-fable
- 2026-09-04T23:04:27Z @neo-fable referenced in commit `ab290ee` - "fix(mcp): an untyped response schema lists as an open object, never as a `result` promise (#310)

`buildOutputZodSchema` wrapped every response schema whose resolved type was not `object` — including schemas with no type at all — while `BaseServer#formatToolResult` wraps only non-object runtime values. For an untyped schema whose runtime value is an object, the listing promised `result`, the payload never carried it, and a strict client rejected every call: `get_instance_properties` (bare schema) and `ingest_source_files` (a `oneOf` composition) were the two instances across the six OpenAPI documents.

The builder now lists a typeless schema — bare or a composition — as an open object, so only a declared non-object type carries the wrapper, the exact set the envelope wraps. `get_instance_properties` declares the property map it already returns (`type: object`, `additionalProperties: true`). A validation spec walks every operation of every server and asserts the listing/envelope agreement, red-first on both instances, plus the untyped and declared-primitive probes."
- 2026-09-04T23:06:08Z @neo-fable cross-referenced by PR #319
- 2026-09-04T23:37:50Z @neo-fable referenced in commit `8efaf6d` - "fix(mcp): a composed response keeps its branches beneath the object root, and the property reader declares its exact shape (#310)

Round 2 of review. A response schema with no top-level type is either a composition or bare. A composition (`oneOf` / `anyOf` / `allOf`) still declares its branches, so the listing keeps them: the root stays `type: object` — the MCP outputSchema contract requires it — and the compiled branches ride beneath it as `anyOf` / `allOf`, so a client rejects a value neither branch declares exactly as it did under the old wrapper. Only a bare schema lists as an open object.

`get_instance_properties` declares what the engine returns — the requested values keyed by name under one `properties` key — rather than an open object. The agreement spec now witnesses both sides: the real `BaseServer#formatToolResult` envelope validated against the emitted listing through the SDK's own Ajv provider, for an object result, a declared primitive, the composed KB ingest contract (two declared shapes accepted, a malformed value rejected) and a bare untyped probe, beside the six-document listing walk."
- 2026-09-04T23:38:30Z @neo-fable referenced in commit `4d3d9cb` - "fix(mcp): an untyped response schema lists as an open object, never as a `result` promise (#310)

`buildOutputZodSchema` wrapped every response schema whose resolved type was not `object` — including schemas with no type at all — while `BaseServer#formatToolResult` wraps only non-object runtime values. For an untyped schema whose runtime value is an object, the listing promised `result`, the payload never carried it, and a strict client rejected every call: `get_instance_properties` (bare schema) and `ingest_source_files` (a `oneOf` composition) were the two instances across the six OpenAPI documents.

The builder now lists a typeless schema — bare or a composition — as an open object, so only a declared non-object type carries the wrapper, the exact set the envelope wraps. `get_instance_properties` declares the property map it already returns (`type: object`, `additionalProperties: true`). A validation spec walks every operation of every server and asserts the listing/envelope agreement, red-first on both instances, plus the untyped and declared-primitive probes."
- 2026-09-04T23:38:30Z @neo-fable referenced in commit `ccea894` - "fix(mcp): a composed response keeps its branches beneath the object root, and the property reader declares its exact shape (#310)

Round 2 of review. A response schema with no top-level type is either a composition or bare. A composition (`oneOf` / `anyOf` / `allOf`) still declares its branches, so the listing keeps them: the root stays `type: object` — the MCP outputSchema contract requires it — and the compiled branches ride beneath it as `anyOf` / `allOf`, so a client rejects a value neither branch declares exactly as it did under the old wrapper. Only a bare schema lists as an open object.

`get_instance_properties` declares what the engine returns — the requested values keyed by name under one `properties` key — rather than an open object. The agreement spec now witnesses both sides: the real `BaseServer#formatToolResult` envelope validated against the emitted listing through the SDK's own Ajv provider, for an object result, a declared primitive, the composed KB ingest contract (two declared shapes accepted, a malformed value rejected) and a bare untyped probe, beside the six-document listing walk."
- 2026-09-04T23:56:06Z @tobiu referenced in commit `442a220` - "Merge pull request #319 from neomjs/agent/310-untyped-output-wrapper

fix(mcp): an untyped response schema lists as an open object, never as a `result` promise (#310)"
- 2026-09-04T23:56:06Z @tobiu closed this issue

