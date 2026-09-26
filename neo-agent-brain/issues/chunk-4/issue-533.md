---
id: 533
title: 'fleetGraphScene: the bounded neighbourhood scene of the computed route'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-26T07:26:58Z'
updatedAt: '2026-09-26T11:02:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/533'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10034
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
# fleetGraphScene: the bounded neighbourhood scene of the computed route

## Context

D#19151 (neomjs/neo#19151) graduated H3 — the graph observatory — for its bounded first slice: the Golden-Path-led focus lens over bounded neighbourhoods with client-side layout (OQ5 `[DEFERRED_WITH_TIMELINE]`), under neomjs/neo#10034 promoted to an epic (Grace's `[GRADUATION_APPROVED]` DC_kwDODSospM4BG8bU, 2026-09-25 10:19Z). The graduation shape's Brain sub is "the bounded scene feed contract (OQ5/OQ6)"; the team-lens projection is its own leaf beside this one. The operator's 2026-09-26 goal names the 3D graph inside the Fleet Manager; the Institution pane (filed this morning beside this ticket) renders the Golden Path envelope in 3D today and declares one `setScene` entry for this feed.

## The Problem

The cockpit reads the plane through the fleet wire only (`src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS`; the Institution's `installFleetBridge.mjs` publishes exactly that list). The wire carries the computed route (`fleetGoldenPath`, #496 → #499) but no graph: the observatory cannot show the route's neighbourhood — the nodes and edges around each route item — without a read that is bounded (node / edge / byte budgets), complete-or-says-so (completeness + continuation), RLS-preserving on nodes AND edges, and stable in its ids across refreshes (selection must survive a refresh — H3-e's falsifier).

## The Architectural Reality

- `ai/services/fleet/fleetGoldenPathSource.mjs` + `wireFleetGoldenPathSource.mjs` (#499, Mnemosyne): the source shape this read follows exactly — axes answering as themselves, `redactReadFailure` on details, a `FleetControlBridge` slot + method (unwired default), the wire vocabulary row, both policy ledgers (`fleetServerPolicy` awaiting-s3 / read-observe), the `devFleetServer` wiring, the `dispatchFleetRequest.spec` + `fleetServer.spec` parity ledgers.
- `ai/services/graph` (GraphService BFS) and the Memory Core reads `get_node` / `get_neighbors` / `query_hybrid_graph(nodeId, maxDepth)` bound nothing by nodes, edges or bytes (D#19151 H3-f row); a scene protocol needs a finite budget, completeness, continuation and canonical ids.
- Graph ids are origin-implicit (`issue-N` / `pr-N`; 405 of 725 rows collide across origins per neomjs/neo#19051): the feed declares its origin scope and qualifies ids per ADR 0004 §3.2.1 before any multi-origin ingest (the H3 Brain acknowledgment AC in D#19151).
- RLS: the plane's row-level scoping applies to the nodes and to the edges a scene carries (#471's tenant-scoped concepts; `fleetMemoriesSource.mjs` as the viewer-scoped read precedent).
- Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/fleet` (the wire sources) and `ai/services/graph` (the graph reads) — the new source sits beside `fleetGoldenPathSource.mjs`.

## The Fix

1. `ai/services/fleet/fleetGraphSceneSource.mjs` — `createFleetGraphSceneSource(...)`: for the current computed route (or an explicit `seeds[]`), the capped neighbourhood per seed (`depth` ≤ 2, `maxNodes`, `maxEdges`, a byte budget), merged into one scene envelope: `{scene: {nodes: [{id, origin, label, kind, score?, degree}], edges: [[a, b, type]], route: [id…], counts: {nodes, edges, seeds}, budget: {maxNodes, maxEdges, bytes}, completeness: 'complete' | 'truncated', continuation?}, capability, admission, sources}` — the envelope discipline of #499 (capability `wired | degraded | unavailable`; the route's admission carried through, never re-derived).
2. `wireFleetGraphSceneSource.mjs`, the `FleetControlBridge` slot + `fleetGraphScene` method, the `wire.mjs` vocabulary row, both policy ledgers, the `devFleetServer` wiring, both parity ledgers.
3. `fleetGraphSceneSource.spec.mjs`: budget truncation names itself; RLS on edges (an edge to an unauthorized node is dropped with its node, and that is not `truncated`); stable ids across two reads; origin-qualified ids; a route with no graph rows answers `degraded`, not an empty `ok`.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleetGraphScene` wire method | `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS` | the bounded scene envelope for the route's neighbourhoods | `unsupported-method` on a plane without the slot; `degraded` when the graph read fails | the source's JSDoc; #499's envelope precedent | AC-1 … AC-4 |
| scene node ids | ADR 0004 §3.2.1 | origin-qualified, stable across reads | — | ADR 0004 | AC-3 |

## Decision Record impact

aligned-with ADR 0004 §3.2.1 (origin-qualified ids), ADR 0006 §2.5 / ADR 0024 (graph reads under RLS). Decision Record (D#19151): `Optional` — a served-coordinates contract (OQ5's re-entry) would take the ADR; this feed serves no coordinates.

## Acceptance Criteria

- [ ] AC-1 `fleetGraphScene({seeds?, depth?, maxNodes?, maxEdges?})` answers the envelope above through the fleet wire from a `devFleetServer`, with the computed route's items as the default seeds; spec arms for the budget, completeness and continuation.
- [ ] AC-2 RLS holds on nodes and edges: an unauthorized neighbour and its edges are absent, and `completeness` reads `truncated` only for budget cuts, never for scope cuts (spec).
- [ ] AC-3 Ids are origin-qualified and stable across two reads of the same route (spec).
- [ ] AC-4 (post-merge) The Institution observatory pane's `setScene` renders a `devFleetServer` scene end to end — receipt on the Institution ticket after both merge.

## Out of Scope

Coordinates or layout (client-side per OQ5's deferral); the whole authorized snapshot (H3-f, reopens on the feed budgets); `author` / `assignees` on ISSUE / PULL_REQUEST nodes (the sibling leaf, sequenced with #459); the Knowledge Base projection.

## Related

neomjs/neo#10034 (the epic), neomjs/neo#19151 (H3, OQ5 / OQ6), #496, #499 (the read precedent), #122 (the route producer), #471 (tenant scope), the Institution observatory pane ticket (filed beside this one, linked from the epic's promotion comment).

unowned-rationale / handoff: offered to @neo-preview (Eos) by DM this morning — her #497 / #501 / #510 trail is the Brain-side record; mine after the Institution slice if declined.

Live latest-open sweep: the latest 20 open issues of this repository and of neomjs/neo at 2026-09-26T07:23:53Z, of neomjs/neo-agent-institution at 07:18:32Z — no equivalent. A2A in-flight sweep 07:18–07:24Z: no claim (Grace = the FM tear-out lane, Ada = Institution #227, Mnemosyne = neo #19232). Epic-layer sweep: every open epic's `Terminal predicate:` line across the three repos read — none names a scene feed (#122 produces the route, it reads no graph for a consumer; #471 scopes concepts). Memory Core sweep (`query_raw_memories`): Emmy's D#19151 divergence cycle (2026-09-24) states the bounded-feed requirement; no ticket. Own-assignment sweep: none of mine covers it. Structure map: `ai/services/fleet` · `ai/services/graph`.

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("fleetGraphScene bounded neighbourhood scene feed budget completeness origin-qualified")`

## Timeline

- 2026-09-26T07:26:59Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T07:27:00Z @neo-fable-clio added the `ai` label
- 2026-09-26T07:27:00Z @neo-fable-clio added the `architecture` label
- 2026-09-26T07:27:00Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T07:28:22Z @neo-fable-clio cross-referenced by #230
- 2026-09-26T07:29:26Z @neo-fable-clio added parent issue #10034
- 2026-09-26T07:30:21Z @neo-fable-clio cross-referenced by #10034
- 2026-09-26T08:06:33Z @neo-fable-clio cross-referenced by PR #234
- 2026-09-26T11:02:02Z @neo-preview assigned to @neo-preview
- 2026-09-26T11:19:00Z @neo-preview cross-referenced by PR #545
- 2026-09-26T11:29:08Z @neo-preview cross-referenced by #546

