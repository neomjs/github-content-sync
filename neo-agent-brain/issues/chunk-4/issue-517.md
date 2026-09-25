---
id: 517
title: removeNodes skips every node the graph cache does not hold
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T21:20:43Z'
updatedAt: '2026-09-25T21:39:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/517'
author: neo-opus-grace
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
  - '[x] 516 The orphan pass forgets by denylist, so every durable label is garbage until named'
blocking: []
---
# removeNodes skips every node the graph cache does not hold

## Context

I found this fixing #511. `GraphService.removeNodes(ids)` calls `Database.removeNode(id)` for each id, which removes the node from the lazy, LRU-bounded cache. SQLite changes only through `onNodesMutate`'s `removedItems` (`ai/graph/Database.mjs:459`). An id the cache doesn't hold removes nothing, so its row and its edges stay in storage. The call returns normally, and its debug line counts the input:
- At 20:46:46Z the local plane logged "Obliterated 126409 Nodes".
- The 19:51Z cycle's GraphLog shows about 2.3k node deletions for a same-sized input.

Observed at L2: #515's spec arm "an orphan loses its vectors only together with its node" evicts one orphan from the cache, and `removeNodes` leaves it in storage.

## The Problem

Six callers assume a storage delete:

| caller | what it assumes | an uncached id |
|---|---|---|
| `GraphMaintenanceService.runGarbageCollection` | orphans leave storage | stays; after #515 its vectors stay too |
| `TemporalSummaryAggregationService.mjs:547` | "graph first, THEN Chroma … never a Chroma-gone / graph-orphaned record" | the Chroma doc goes and the graph node stays, which is the record it promises never to leave |
| `IssueIngestor.mjs:172` | removed issues leave the graph | the node stays while its vectors are deleted |
| `purgeNoContentGraphMemories.mjs:218` | the plan's nodes are deleted | the plan is read straight from SQLite (`buildCleanupPlan({db})`), so in the script's own process those nodes are uncached, yet it reports `deletedNodes = plan.nodeIds.length` |
| `MemoryService.mjs:1601` | the archived-identity node is gone | stays |
| `nlActionTelemetryStore.mjs:228` | expired telemetry is removed | accumulates |

The error runs in the safe direction (rows kept, not lost), but every caller's own report is wrong, and the maintenance purge certifies deletions that never happened.

## The Architectural Reality

- `ai/graph/Database.mjs`: `removeNode` works on the cache; `onNodesMutate` → `storage.removeNodes` (`:459`).
- `ai/graph/storage/SQLite.mjs:560`: `removeNodes`, the row delete that `neomjs/neo#11140` put behind the destructive-operation guard.
- Edges reference `Nodes(id)` with `ON DELETE CASCADE` under `foreign_keys=ON`, so a node row's delete takes its edge rows with it.

## The Fix

When the cache lacks the node and storage is attached with `autoSave`, `Database.removeNode` deletes the stored row through `storage.removeNodes`, the same guarded path the mutate branch uses. `GraphService.removeNodes`' debug line counts the rows that left storage.

## Sequencing

This must not land while the orphan pass's candidate set is unbounded. On today's plane that set is about 126k nodes, most of them edgeless because #506 severed their edges, and a storage-exact `removeNodes` would delete them in one cycle.
- **Blocked by #516,** which bounds the set to `CONCEPT`.
- **Even then,** the local plane held 28,836 edgeless `CONCEPT`s at 21:10Z, mostly #506 victims rather than faded ones. REM does not re-digest a digested session, so a concept deleted now is gone. This also waits for the plane's edge-recovery decision (#509's lane): restore those edges first, or accept that forgetting.

## Acceptance Criteria

- [ ] `removeNodes` deletes an uncached node's row and its edge rows from storage (red on `dev`).
- [ ] A cached node's removal is unchanged, and the destructive-operation guard covers both branches.
- [ ] `purgeNoContentGraphMemories --apply` reports the rows that left storage, not the plan's size.

## Out of Scope

- **Which labels the orphan pass may collect.** That is #516.
- **Re-running the maintenance purge on any plane.** That is an operator step once this lands.

## Related

#511 · #515 · #506 · `neomjs/neo#11140` · `neomjs/neo#11698`

Decision Record impact: none.
Live latest-open sweep: checked latest 20 open issues at 2026-09-25T21:18:39Z; no equivalent found.
A2A in-flight sweep: last 12 messages at 21:19Z; no claim on this scope.
MC sweep: "removeNodes does not delete nodes from SQLite when not cached; purge reports deleted nodes but they remain in the graph", 6 results; prior art `neomjs/neo#11140` (the guard this fix routes through), no prior decision on the cache branch.
Own-assignment sweep: 5 open, #511 adjacent (this is its named follow-up).

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201
Retrieval Hint: "removeNodes uncached ids storage delete Database.removeNode onNodesMutate"


## Timeline

- 2026-09-25T21:20:43Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T21:20:44Z @neo-opus-grace added the `bug` label
- 2026-09-25T21:20:45Z @neo-opus-grace added the `ai` label
- 2026-09-25T21:20:45Z @neo-opus-grace added the `agent-os` label
- 2026-09-25T21:21:00Z @neo-opus-grace marked this issue as being blocked by #516
- 2026-09-25T21:21:17Z @neo-opus-grace cross-referenced by #516
- 2026-09-25T21:21:18Z @neo-opus-grace cross-referenced by #511
- 2026-09-25T21:51:49Z @neo-opus-vega cross-referenced by PR #520
- 2026-09-25T21:52:38Z @neo-opus-vega cross-referenced by #521

