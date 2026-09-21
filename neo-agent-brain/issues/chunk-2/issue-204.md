---
id: 204
title: Expose a verified drag gesture through Neural Link
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - model-experience
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-27T21:14:05Z'
updatedAt: '2026-08-30T12:25:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/204'
author: neo-gpt
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 17821 Add atomic drag execution to the Neural Link client'
blocking: []
closedAt: '2026-08-30T12:25:34Z'
---
# Expose a verified drag gesture through Neural Link

## Context

Neural Link currently exposes `simulate_event` as its generic write-locked event channel. The Brain-side OpenAPI description says it can be used for drag sequences, but the caller must supply every DOM id, window id, event type, coordinate, and delay, and the response is only a Boolean.

Live source verification on Brain `dev` found:

- `ai/mcp/server/neural-link/openapi.yaml` owns the `simulate_event` request and response schema and classifies it as `write-locked`.
- `ai/services/neural-link/InteractionService.mjs#simulateEvent()` forwards the raw event array to the Engine client without a gesture-level contract.
- The Brain structure map places this forwarding service beside the existing drag observation methods under `ai/services/neural-link/`.
- Knowledge Base and live GitHub searches found routing, raw-recipe, drag-observability, and semantic-dock-operation tickets, but no first-class verified drag gesture tool.

The Engine dependency is [neomjs/neo#17821](https://github.com/neomjs/neo/issues/17821). It owns atomic browser execution, DOM/window geometry, Mouse-sensor arming, cleanup, and the physical lifecycle receipt.

## The Problem

An agent that wants to drag a splitter, grid header, tab, or cross-window surface must currently reconstruct Engine internals from raw events. The tool can report success without proving `drag:start`, movement, or `drag:end`, and its caller must separately discover the current arming thresholds.

The Brain needs a stable agent-facing gesture contract, but it must not absorb DOM access, coordinate conversion, timing thresholds, or drag implementation from Engine. Without that split, every Engine sensor change can silently invalidate the MCP tool.

## The Architectural Reality

- Brain owns the public MCP/OpenAPI contract, tool-tier metadata, server-side argument validation, and forwarding service.
- Engine owns the browser-side `drive_drag` RPC and returns one typed outcome envelope for both success and failure. Brain passes the request through; it never builds mouse-event arrays, resolves DOM geometry, or copies sensor thresholds.
- Current JSON-RPC error transport is not a receipt channel: Engine Client sends only message/stack and Brain ConnectionService reconstructs an Error from message only. Brain therefore consumes Engine `success:false` as a normal RPC result and converts it at the MCP boundary.
- `BaseServer#formatToolResult()` already maps object results with an `error` key to `isError:true`, but currently drops structuredContent for that branch. This ticket extends that existing formatter so the exact error object—including `phase` and `receipt`—remains machine-readable.
- Existing `get_drag_state`, `get_drag_trace`, `observe_motion`, and `verify_component_consistency` remain independent read tools. They verify consumer-specific outcomes; they do not actuate a gesture.
- Existing `execute_dock_operation` remains the semantic dock-model path. It must not replace a physical drag when the behavior under test is the input pipeline.
- The tool remains `write-locked`, consistent with Brain #143.
- No new service class is required: extend the existing OpenAPI spec, InteractionService, BaseServer formatter, and existing validation/test surfaces.

## The Fix

Add `/interaction/drive_drag` with `operationId: drive_drag`, `x-neo-tool-tier: write-locked`, and `x-pass-as-object: true`.

Its request mirrors the corrected Engine #17821 contract exactly:

- required `source`: DOM `targetId`, `windowId`, optional normalized anchor;
- required `destination`: exactly one node, target-local point, or source-relative delta mode;
- optional node/target-local-point `waypoints`;
- `steps`: post-arm moves across the whole polyline, integer 1..120;
- optional `durationMs`: post-arm duration, at least `steps * 16` and at most 30000;
- optional `sessionId` for explicit App-worker targeting;
- no caller-supplied screen coordinates and no destination-window event routing.

`InteractionService#driveDrag()` forwards the validated object as Engine RPC `drive_drag`. It contains no Mouse delay, minimum-distance, DOM, coordinate-conversion, interpolation, or cleanup logic.

Engine returns:

- `success:true` with the physical receipt; Brain returns that receipt unchanged.
- `success:false` with `phase`, `error:{code,message}`, and partial `receipt`; Brain maps it to an object error result:
  `{error:'Drag gesture failed', message, phase, receipt}`.

`BaseServer#formatToolResult()` preserves that full object as `structuredContent` while setting `isError:true`. The text block remains concise; machines receive the exact partial receipt. No Error custom-property transport and no JSON embedded in a message.

Existing `simulate_event` remains the raw event-sequence escape hatch and never delegates to `drive_drag`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| New MCP tool `drive_drag` | This ticket + Engine #17821 | One write-locked call requests a whole physical Neo drag and returns Engine's typed outcome | Validation rejects ambiguous modes/bounds; no raw-event fallback | OpenAPI description | OpenAPI schema positive/negative matrix + live call |
| `InteractionService#driveDrag()` | Existing neural-link InteractionService | Forward the validated object to Engine RPC and preserve success; convert Engine typed failure to object error | Missing/incompatible Engine method fails clearly | JSDoc | Focused service unit exact payload + success/failure |
| MCP structured error result | `BaseServer#formatToolResult()` | Object with `error` yields `isError:true` and retains full object as `structuredContent` | Primitive/thrown errors keep existing behavior | BaseServer JSDoc | Formatter unit proves phase/receipt survive |
| Existing `ConnectionService` | Current RPC transport | Receives Engine typed outcome as ordinary result; no custom Error metadata transport required | Network/unknown-method errors remain ordinary RPC errors | Existing docs | Connection service unit |
| Existing `simulate_event` | Current OpenAPI + service | Remains raw sequence control | No deprecation or silent delegation | Existing docs | Existing contract green |
| Existing drag read tools | Current InteractionService/OpenAPI | State/trace/motion/consistency remain independent | No hidden mandatory recorder | Existing docs | Existing tool tests + post-gesture composition witness |
| Tool tier | Existing tier policy + Brain #143 | `drive_drag` is `write-locked` | Absent from read-only projections | OpenAPI metadata | Capability-matrix positive/negative controls |

## Decision Record impact

`aligned-with ADR 0020 §3` — Neural Link remains the bridge; the Brain exposes the capability while Engine owns browser execution.

## Acceptance Criteria

- [ ] OpenAPI defines `drive_drag` with the exact Engine source plus discriminated node / target-local-point / source-delta destination; ambiguous modes, invalid anchors, and bounds reject.
- [ ] The tool is `write-locked`, object-passed, and absent from read-only projections.
- [ ] `InteractionService#driveDrag()` forwards the request to Engine RPC `drive_drag` without copying thresholds, resolving geometry, constructing events, or routing by destination window.
- [ ] Engine `success:true` preserves the physical receipt fields and claims no semantic application outcome.
- [ ] Engine `success:false` becomes MCP `isError:true` with machine-readable `structuredContent:{error,message,phase,receipt}`.
- [ ] `BaseServer#formatToolResult()` retains structured content for object error results; existing success, primitive, policy-refusal, and thrown-error paths remain green.
- [ ] A missing Engine method fails clearly; Brain never falls back to a hand-built `simulate_event` recipe.
- [ ] Existing `simulate_event` and drag observation tools retain their contracts.
- [ ] Integration against Engine #17821's merged/current dev drives one same-window splitter or draggable and returns correlated start/move/end truth; a missing-target red control returns the exact resolution/receipt failure.
- [ ] A separate consumer semantic assertion follows the physical receipt, proving gesture completion is not conflated with application commit.
- [ ] Cross-family review is required before merge.

## Out of Scope

- DockLayouts architecture, any Ideation Sandbox, grid/dock reducers, splitter feature behavior, or application-specific semantic assertions.
- Implementing browser geometry, Mouse thresholds, event construction, or gesture cleanup in Brain.
- Removing, renaming, or deprecating `simulate_event`.
- Migrating learning guides between repositories; Brain issue [#10](https://github.com/neomjs/neo-agent-brain/issues/10) owns that custody move. This ticket documents the tool in OpenAPI/JSDoc only.
- Bulk migration of existing manual drag recipes, replay ledgers, diagnostics, or proof-only infrastructure.

## Avoided Traps

- **Brain as a second drag engine:** rejected; it forwards one object and interprets one outcome.
- **Thrown Error receipt across JSON-RPC:** rejected; both current transport layers discard custom metadata.
- **Success:false returned as a green MCP result:** rejected; the MCP formatter marks the object error `isError:true` and preserves structuredContent.
- **JSON receipt embedded in error message:** rejected; structured data stays structured.
- **Semantic dock-only tool:** rejected; splitters, grids, tabs, and generic draggables share the physical requirement.
- **Raw-event fallback on an old Engine head:** rejected; it would make one MCP call mean different things across clients.
- **Bundling consumer rewrites:** rejected; one MCP contract and one integration witness.

## Related

- BLOCKED_BY [neomjs/neo#17821](https://github.com/neomjs/neo/issues/17821)
- Agent-control surface context: [#141](https://github.com/neomjs/neo-agent-brain/issues/141)
- Tool-tier authority: [#143](https://github.com/neomjs/neo-agent-brain/issues/143)
- Learning-guide custody: [#10](https://github.com/neomjs/neo-agent-brain/issues/10)
- Historical Engine work: [neomjs/neo#12884](https://github.com/neomjs/neo/issues/12884), [neomjs/neo#12807](https://github.com/neomjs/neo/issues/12807), [neomjs/neo#12886](https://github.com/neomjs/neo/issues/12886), [neomjs/neo#14587](https://github.com/neomjs/neo/issues/14587)

Live latest-open sweep: checked the latest 20 open issues in both Brain and Engine plus exact `drive_drag` / verified-drag / `simulateDrag` / drag-gesture searches at `2026-08-27T21:13:49Z`; no equivalent found beyond the newly created Engine dependency #17821. A2A all-status recency sweep: 30 messages; no competing Neural Link drag claim.

Origin Session ID: `db682c7c-7b7a-4e42-8374-71944a42db40`

Retrieval Hint: "Brain Neural Link drive_drag write-locked MCP verified physical gesture Engine dependency"



## Timeline

- 2026-08-27T21:14:06Z @neo-gpt added the `enhancement` label
- 2026-08-27T21:14:06Z @neo-gpt added the `ai` label
- 2026-08-27T21:14:07Z @neo-gpt added the `testing` label
- 2026-08-27T21:14:07Z @neo-gpt added the `model-experience` label
- 2026-08-27T21:14:07Z @neo-gpt added the `agent-os` label
- 2026-08-27T21:14:27Z @neo-gpt cross-referenced by #17821
- 2026-08-29T20:37:21Z @neo-gpt cross-referenced by PR #17862
- 2026-08-30T04:44:55Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-30T05:00:23Z @neo-gpt-emmy cross-referenced by PR #249
- 2026-08-30T07:19:25Z @neo-gpt-emmy referenced in commit `06af003` - "feat(neural-link): expose verified drag gestures (#204)"
- 2026-08-30T07:19:26Z @neo-gpt-emmy referenced in commit `ab1a04d` - "test(memory-core): pin strict community envelopes (#204)"
- 2026-08-30T12:25:34Z @tobiu referenced in commit `d7090f9` - "Merge pull request #249 from neomjs/codex/204-drive-drag

feat(neural-link): expose verified drag gestures (#204)"
- 2026-08-30T12:25:34Z @tobiu closed this issue

