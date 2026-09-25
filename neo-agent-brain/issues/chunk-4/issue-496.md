---
id: 496
title: The Fleet Manager reads the computed Golden Path through one fleet-wire method
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-25T15:48:45Z'
updatedAt: '2026-09-25T19:20:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/496'
author: neo-fable
commentsCount: 0
parentIssue: 122
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T19:20:54Z'
---
# The Fleet Manager reads the computed Golden Path through one fleet-wire method

## Context

The operator's refocus of 2026-09-25 (15:42Z, his words: "a re-focus to address the high ROI lanes first. e.g. GP and graph inside FM") and the lead's lane 3 ("a GP text representation in the Fleet Manager") ask for the Golden Path to be visible inside the cockpit. The Golden Path already has a typed text artifact: `GoldenPathSynthesizer` writes `computed-route.json` (the computed-route.v1 sidecar: `capturedAt`, `expiresAt`, `status`, `freshness`, `provenance`, `route.items`) beside the handoff, and `AgentOrchestrator#readComputedRoute` already reads it fail-closed. The corpus-projection admission (`corpusProjectionContract.mjs`) already says whether that route is current, last-known-good, or withheld; today it reads withheld (`CORPUS_PROJECTION_NOT_CURRENT`, `freshness-sla-breached`; REM 990 undigested, 0 recent cycles). Nothing serves either to the cockpit: the fleet wire (`src/fleet/contract/wire.mjs`) has `fleetTasks`, `fleetActivity`, `fleetMemories` and their siblings, and no Golden Path read.

## The Problem

The cockpit can render the roster, activity, tasks and catch-up envelopes because each has one bridge read backed by one wired source with an honest not-wired default. The Golden Path has none, so the one picture the institution steers by is invisible in the product, and any pane built without a read would have to guess at freshness. The read must carry the same honest states as `get_context_frontier` (current / last-known-good with its age / withheld with the reason) — never a stale route presented as current, never a mixed live read.

## The Architectural Reality

- `ai/services/fleet/FleetControlBridge.mjs`: source slots (`tasksSource`, `activitySource`, …) and READ-OBSERVE methods (`fleetTasks`, ~:628) returning the source envelope untouched or a `capability: {state: 'unavailable', reason: '… not wired'}` default.
- `ai/services/fleet/fleetTasksSource.mjs` + `wireFleetTasksSource.mjs`, `wireFleetActivityReadSource.mjs`: the source-module + boot-wiring precedent (read at the use site, fail-soft, no stub); `devFleetServer.mjs` (~:147, ~:299) is the boot site.
- `src/fleet/contract/wire.mjs`: `FLEET_WIRE_METHODS` (the public vocabulary, protocol version 1, `method-schema-v1`); `ai/services/fleet/fleetServerPolicy.mjs`: the per-method policy (`read-observe`) and the S3 gating map.
- `ai/agent/AgentOrchestrator.mjs#readComputedRoute` (~:117): the sidecar read with `validateComputedRouteResult`, freshness and expiry gates.
- `ai/services/graph/corpusProjectionContract.mjs`: `CORPUS_PROJECTION_CONSUMER.computedGoldenPath`, `evaluateCorpusProjectionAdmission`, `readCorpusProjectionReceipt`; `MemoryService.mjs` (~:119): the withheld / last-known-good view shape.
- Structure map (`npm run ai:structure-map -- --files --loc`, 2026-09-25 15:47Z): owning directories `ai/services/fleet` and `ai/services/graph`.

## The Fix

1. `ai/services/fleet/fleetGoldenPathSource.mjs`: `createFleetGoldenPathSource({routePath, readReceipt, config, now, readRemState})` → `readGoldenPath(params)` returning one envelope: `{capability, admission, route, rem, generatedAt}`. `route` is the validated sidecar passed through (`status`, `freshness`, `capturedAt`, `expiresAt`, `provenance`, `route.kind`, `route.items[]` — id, title, score, reasons as the producer wrote them), never re-ranked; an unreadable or contract-invalid sidecar is `capability.state: 'degraded'` with the reason; `admission` is the consumer's projection admission (`admitted`, `fallback`, `reasonCode`, `staleFacets`); `rem` is `{undigested, digested, recentCycles}` when readable.
2. `wireFleetGoldenPathSource.mjs` installs it at the fleet-server boot (the `wireFleetTasksSource` shape), reading the handoff directory from config at the use site.
3. `FleetControlBridge#fleetGoldenPath(params)` over a `goldenPathSource` slot with the unwired default `{capability: {state: 'unavailable', reason: 'fleet golden path source not wired'}, admission: null, route: null, rem: null}`; `fleetGoldenPath` joins `FLEET_WIRE_METHODS` and the server policy as `read-observe`.
4. Unit arms: the source over a fixture sidecar (fresh / expired / invalid / missing) and a fixture receipt (admitted / stale), the bridge's unwired default, the wire vocabulary + policy parity.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleetGoldenPath` (fleet wire method) | `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS`; `fleetServerPolicy.mjs` | READ-OBSERVE; returns the envelope above for the authenticated viewer | unknown method fails closed as today | this ticket + the module JSDoc | unit arm on vocabulary/policy parity |
| `FleetControlBridge#goldenPathSource` (slot) | the bridge's DI contract (`tasksSource` precedent) | wired at boot; unwired → `capability.state 'unavailable'` | the honest not-wired envelope | module JSDoc | bridge unit arm |
| envelope `route` | computed-route.v1 (`validateComputedRouteResult`) | pass-through, validated; non-fresh routes carry their own `status`/`freshness` | `capability.state 'degraded'` + reason | module JSDoc | source unit arms |
| envelope `admission` | `corpusProjectionContract` consumer `computed-golden-path` | current / last-known-good / withheld, as the contract evaluates | `admitted: false`, `fallback 'last-known-good'` | existing contract docs | source unit arm |

Decision Record impact: aligned-with ADR 0023 (earned-and-forgetting map fidelity: the read presents the producer's route and its freshness, adds no scorer, no age multiplier, no synthesis).

## Acceptance Criteria

- [ ] AC-1 `fleetGoldenPath` answers over the wire from a running fleet server with the envelope above; against today's plane it reports the withheld admission and the last route's `capturedAt`, not a current picture.
- [ ] AC-2 The source's unit arms: fresh sidecar → `route.status 'fresh'` passed through; expired / invalid / missing → `capability.state 'degraded'` with the reason and no items; admitted vs stale receipt → the matching `admission`.
- [ ] AC-3 The bridge's unwired default and the wire vocabulary / server-policy parity are unit-tested; `lint-openapi-service-parity` and the fleet contract's own specs stay green.
- [ ] AC-4 No ranking, merging or synthesis in the read path (reviewed against the module diff).

## Out of Scope

The cockpit pane that renders the envelope (neomjs/neo-agent-institution, filed alongside); the REM/dream currency itself (#64, #495); concept-graph rendering inside the cockpit (Institution #8's COP).

## Related

#122 (parent: Golden Path v2 — consumers over a measured route), #64 (the heavy-maintenance lease starving REM), #495, the Institution pane ticket (linked from the broadcast), neomjs/neo-agent-institution#9 / #10.

Live latest-open sweep: checked the latest 20 open issues of this repository and of neo-agent-institution at 2026-09-25T15:47:01Z; no equivalent. A2A in-flight sweep (the 12 most recent messages, all read states, 15:44Z) and my [lane-intent] broadcast at 15:46Z: no competing claim. Memory Core sweep: no prior record of a Golden Path read for the cockpit. Own-assignment sweep: none of mine covers it.

Retrieval Hint: `query_raw_memories("fleetGoldenPath wire method computed route envelope cockpit")`

Origin Session ID: 4c0a5550-17ba-4752-9852-846afa537c86

## Timeline

- 2026-09-25T15:48:45Z @neo-fable assigned to @neo-fable
- 2026-09-25T15:48:46Z @neo-fable added the `enhancement` label
- 2026-09-25T15:48:47Z @neo-fable added the `ai` label
- 2026-09-25T15:48:47Z @neo-fable added the `agent-os` label
- 2026-09-25T15:49:20Z @neo-fable added parent issue #122
- 2026-09-25T15:58:34Z @neo-fable cross-referenced by PR #499
- 2026-09-25T16:43:53Z @neo-opus-vega cross-referenced by #500
- 2026-09-25T17:01:29Z @neo-preview cross-referenced by PR #502
- 2026-09-25T17:06:04Z @neo-fable referenced in commit `2d7e916` - "fix(fleet): the golden path source reads its projection leaves at the use site (#496)

The wiring passed AiConfig.orchestrator.corpusProjection through as a config-shaped object (ADR 0019 B5, flagged by lint-config-template-ssot). The source now reads enabled / receiptPath / sourceRepository / sourceRef inline where the admission is evaluated; the factory takes an injectable readAdmission seam instead of a config subtree, and the spec drives it through that seam. Lint OK locally; fleet specs unchanged (111/112, the remRunStateDir boot-gate fixture stays the pre-existing red)."
- 2026-09-25T17:41:16Z @neo-preview cross-referenced by #503
- 2026-09-25T18:56:22Z @neo-opus-vega cross-referenced by PR #215
- 2026-09-25T19:03:10Z @neo-fable referenced in commit `e4059d2` - "fix(fleet): the golden path route crosses the plane boundary through get_computed_route (#496)

RA-2: the packaged shell runs the fleet server on the host in plane-attach mode, and the synthesizer's computed-route.json lives on the plane's handoff volume, so a host file read answered route-sidecar-missing by construction. A new Memory Core operation, get_computed_route (the get_sandman_handoff sibling: readComputedRoute is the file-contract half, the tool binds the resolved handoff path and reads the projection leaves inline), serves the validated sidecar with the computed-golden-path admission from the process that mounts the volume; the fleet source reads it through the same operation boundary as the REM state, so the envelope's three axes are one plane's. RA-1: the source spec, the dispatch ledger spec and the store spec join the unit job's run list. Ledger completion found by the run: the compliance tier map lacked the three community operations the OpenAPI declares (fails identically on dev)."
- 2026-09-25T19:20:54Z @tobiu referenced in commit `3655139` - "Merge pull request #499 from neomjs/feat/496-fleet-golden-path-read

feat(fleet): the cockpit reads the computed Golden Path through one wire method (#496)"
- 2026-09-25T19:20:54Z @tobiu closed this issue
- 2026-09-25T19:23:26Z @neo-fable cross-referenced by #122

