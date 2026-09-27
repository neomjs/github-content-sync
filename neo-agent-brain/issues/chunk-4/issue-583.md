---
id: 583
title: 'fleetGraphScene serves the whole authorized graph, with the route as its overlay'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T14:28:35Z'
updatedAt: '2026-09-27T16:28:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/583'
author: neo-opus-ada
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
closedAt: '2026-09-27T16:28:25Z'
---
# fleetGraphScene serves the whole authorized graph, with the route as its overlay

## Context

This is the live input for the FM goal "the Graph inside FM". Institution #291 (Grace, approved, resolves neomjs/neo-agent-institution#288) draws the whole graph in communities with the route as an overlay, and proves it with a 100k-node fixture. The live input is D#19151's open row (neomjs/neo#19151). The plane holds 243,775 nodes and 233,563 edges (a read-only count on 2026-09-27).

## The Problem

`fleetGraphScene` answers only the route's neighbourhood. It walks `get_node` / `get_neighbors` with one Memory Core call per node, caps the result at 150 nodes / 300 edges / 32 KiB, and answers `scene: null` when there is no route. The Memory Core has no bulk graph read. `GraphService.listNodeRecordsByType` / `listEdgeRecordsByType` are in-process, work per type, and return raw property bags. So the Fleet cannot ask for the graph.

## The Fix

1. `GraphService.readSceneGraph({maxNodes, maxEdges})`: one RLS-scoped read. It uses the SQL predicate of `listNodeRecordsByType` plus the `isRlsVisible` recheck, returns display fields only (`{id, kind, label}`) and the visible edges whose endpoints are both in the returned node set, and flags truncation per list.
2. The Memory Core operation `get_graph_scene` (read tier) serves that read.
3. `fleetGraphSceneSource` reads the scene through that one operation, qualifies the ids, and carries the computed route as `route` (the overlay). A missing route gives `route: []` and still serves the graph. The per-node walk and its `get_node` / `get_neighbors` seams retire. The default budget holds today's graph whole, and a cut reads `truncated`.
4. `devFleetServer` wires `get_graph_scene`.

## Contract Ledger Matrix

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_graph_scene` → `GraphService.readSceneGraph` | `ai/mcp/server/memory-core/openapi.yaml` | Columnar `{kinds, types, nodes: {ids, kinds, labels}, edges, counts: {nodes, edges, unlinked}, budget: {maxNodes, maxEdges}, truncated: {nodes, edges}}` under RLS on both tables. Budgets default to 250k / 500k, and a node cut keeps the best-connected | Without the SQLite store it throws, never answering the node cache | the method's JSDoc, the operation's description | AC-2 |
| `fleetGraphScene` envelope | the wire method (`src/fleet/contract/wire.mjs`), the FM's `GraphSceneEnvelope` shape | `{capability, scene: {route, nodes: [{id, label, kind}], edges: [{from, to, type?}], counts: {nodes, edges, seeds, unlinked}, budget: {maxNodes, maxEdges, maxBytes}, completeness}, snapshotId, capturedAt}` | A failed or malformed graph read is `unavailable`; an empty one is `degraded`; an unreadable route gives `route: []` | the source's module doc | AC-1, AC-3 |
| `counts` (both layers) | this ticket | The counts describe the lists delivered. `unlinked` is the delivered nodes no delivered edge names, recounted after every budget or byte cut. The FM's closed shape ignores `unlinked` until a layout adopts it | — | both JSDocs | the byte-cut and edge-budget arms |

## Acceptance Criteria

- [ ] AC-1 `fleetGraphScene({})` answers the authorized graph in the existing scene shape, which the FM's `GraphSceneEnvelope` lands unchanged, with the route as `route`. A routeless plane still gets its graph (spec).
- [ ] AC-2 RLS: a node the viewer may not see is absent along with every edge touching it, and that absence is not `truncated`. Only a budget cut reads `truncated` (spec over a real SQLite GraphService).
- [ ] AC-3 Ids are origin-qualified, and two reads of one graph give byte-equal scenes (spec).
- [ ] AC-4 The PR records the node and edge counts, the bytes and the read time of the read's logic run read-only against the live plane's graph. The operation is not deployed before merge.

## Post-Merge Validation

- One `get_graph_scene` call on the deployed plane answers within the recorded counts and time.
- The installed FM's Observatory draws the live graph after the Brain pin bump and repackage.

Decision Record impact: aligned with ADR 0004 §3.2.1 (origin-qualified ids) and ADR 0006 §2.5 / ADR 0024 (graph reads under RLS).

Related: #533 (the neighbourhood read this replaces) · neomjs/neo-agent-institution#288 · neomjs/neo#19151

Live latest-open sweep (2026-09-27T14:30Z): the open issues of this repository and the Institution matching graph / snapshot / scene / observatory. #533 puts the whole snapshot out of scope, and neomjs/neo-agent-institution#288 is the renderer. No open Brain PR touches the scene feed. Ownership: Emmy's confirmation (A2A, 13:59Z).

Authored by Ada (Claude Opus 5.5, Claude Code). Session f3d50317-fe3b-4773-b4ac-db05e1fa6812.


## Timeline

- 2026-09-27T14:28:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T14:28:36Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T14:28:36Z @neo-opus-ada added the `ai` label
- 2026-09-27T14:52:45Z @neo-opus-vega cross-referenced by #585
- 2026-09-27T15:07:39Z @neo-opus-ada cross-referenced by PR #587
- 2026-09-27T15:09:54Z @neo-opus-vega cross-referenced by PR #588
- 2026-09-27T15:20:40Z @neo-opus-ada referenced in commit `a212139` - "feat(fleet): fleetGraphScene serves the whole graph the viewer may see, with the route as its overlay (#583)

GraphService.readSceneGraph reads both tables in keyset pages, yielding between
pages, under the SQL RLS clause plus the isRlsVisible recheck, and answers every
visible node and relation in columnar display fields; counts.unlinked names the
nodes with no visible relation, which a budget cut drops first. The Memory Core
serves it as get_graph_scene. fleetGraphSceneSource reads it in one call,
qualifies ids, and carries the computed route as the overlay, so a missing route
no longer withholds the graph. The per-node get_node / get_neighbors walk
retires, and the byte trim is linear."
- 2026-09-27T15:28:40Z @neo-gpt cross-referenced by PR #291
- 2026-09-27T16:03:18Z @neo-opus-ada referenced in commit `a0e98a5` - "fix(fleet): a scene's counts describe the lists it delivers, unlinked included, after every cut (#583)

Emmy's falsifier at a212139: four edge-less nodes byte-cut to two still read
unlinked 4, because the trim recounted nodes and edges but not unlinked, and the
Memory Core counted unlinked by pre-budget degree. countScene now recounts all
three from the delivered lists after projection and after every trim, and the
Memory Core counts the answered nodes no answered edge names."
- 2026-09-27T16:05:55Z @neo-preview cross-referenced by PR #592
- 2026-09-27T16:19:03Z @neo-opus-ada referenced in commit `98e5007` - "feat(fleet): fleetGraphScene serves the whole graph the viewer may see, with the route as its overlay (#583)

GraphService.readSceneGraph reads both tables in keyset pages, yielding between
pages, under the SQL RLS clause plus the isRlsVisible recheck, and answers every
visible node and relation in columnar display fields; counts.unlinked names the
nodes with no visible relation, which a budget cut drops first. The Memory Core
serves it as get_graph_scene. fleetGraphSceneSource reads it in one call,
qualifies ids, and carries the computed route as the overlay, so a missing route
no longer withholds the graph. The per-node get_node / get_neighbors walk
retires, and the byte trim is linear."
- 2026-09-27T16:19:03Z @neo-opus-ada referenced in commit `893ef27` - "fix(fleet): a scene's counts describe the lists it delivers, unlinked included, after every cut (#583)

Emmy's falsifier at a212139: four edge-less nodes byte-cut to two still read
unlinked 4, because the trim recounted nodes and edges but not unlinked, and the
Memory Core counted unlinked by pre-budget degree. countScene now recounts all
three from the delivered lists after projection and after every trim, and the
Memory Core counts the answered nodes no answered edge names."
- 2026-09-27T16:28:25Z @tobiu referenced in commit `81b75b2` - "Merge pull request #587 from neomjs/ada/583-graph-scene

feat(fleet): fleetGraphScene serves the whole graph the viewer may see, with the route as its overlay (#583)"
- 2026-09-27T16:28:26Z @tobiu closed this issue

