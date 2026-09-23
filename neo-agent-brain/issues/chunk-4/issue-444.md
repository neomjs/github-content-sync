---
id: 444
title: 'Summary discovery re-scans the graph per memory row, ~50 min per run'
state: OPEN
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T15:11:25Z'
updatedAt: '2026-09-23T15:11:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/444'
author: neo-opus-vega
commentsCount: 0
parentIssue: 64
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
# Summary discovery re-scans the graph per memory row, ~50 min per run

## Context

On the local plane (images `b99ea11`; `SessionService.mjs` is identical at `dev` `ef13cdb`), two summary runs sat in drift detection for most of their heavy-maintenance lease hold before logging `Found N sessions`: 12:38:27 → 13:32:38Z (54 min) on 2026-09-23, and the run started at 14:22:44Z, still in the same phase 45 minutes later with the `summarize-sessions.mjs` child at 85 % CPU and 652 MB RSS. Neither the metadata scan (~2 s) nor receipt recovery (no replays) explains the gap. It is CPU, not I/O. This is the unattributed half of the hold whose other half is #438.

## The Problem

`findSessionsToSummarize` (`SessionService.mjs:469`) calls `getExternallyActiveSessionIds` (`:240`) once per drift sweep. That method runs one synchronous better-sqlite3 statement over `Nodes` whose correlated `EXISTS` re-scans the whole table for `WAKE_SUBSCRIPTION` rows for every memory row. `EXPLAIN QUERY PLAN` on the plane's graph reads `SCAN memory | SCAN subscription EXISTS`, and `Nodes` has only its primary-key autoindex, so every predicate is a `json_extract` over the full table.

Measured read-only on the plane's `memory-core-graph.sqlite` (230,740 nodes, 33,425 memory rows with a session, 17 `WAKE_SUBSCRIPTION` nodes, 12 subscribed identities):

| shape | rows | time |
|---|---|---|
| today's correlated `EXISTS` | 50 memory rows | 2,793 ms, a lower bound of ~31 min for all 33,425 (a row whose identity has no subscription scans the whole table) |
| identities first (one scan) | 12 identities | 345 ms |
| memories `IN (…)` those identities (one scan) | 30,776 rows | 401 ms |

The statement is synchronous, so the child's event loop is blocked for the duration: no log line, no progress, the lease held throughout.

## The Fix

In `getExternallyActiveSessionIds`, resolve the subscribed identities once (the same `WAKE_SUBSCRIPTION`, active-status and `harnessTarget` predicates), return early when there are none, and select memory rows whose identity is `IN` that list. The result set and the rest of the method stay unchanged.

Not in scope: deleting the check. The call site calls it "largely subsumed by the idle-gate above … pending the follow-up cleanup", but deleting it changes the outcome for memories whose Chroma metadata has no timestamp. That follow-up stays with the comment.

## Acceptance Criteria

- [ ] **AC-1** `getExternallyActiveSessionIds` returns the same set as today: the existing arms in `QueryReRanker.spec.mjs` pass unchanged.
- [ ] **AC-2** The method no longer scans `Nodes` once per memory row. `EXPLAIN QUERY PLAN` of its statements shows no correlated `EXISTS` scan, and on a read-only copy of the plane's graph the call completes in seconds.
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* After deploy, a summary run's `Starting drift-detection` → `Found N sessions` gap is seconds, not tens of minutes. Residual-Owner: #64.

## Related

#64 (parent: scheduling starvation) · #438 (the same hold's timeout half) · neomjs/neo#13637 (the churn gate beside it)

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:39Z plus those filed since (#440–#443), none equivalent. Org keyword sweeps (`getExternallyActiveSessionIds`, `externally active session`, `WAKE_SUBSCRIPTION json_extract`) return only closed predecessors: the churn gate and the discovery memory bound (neomjs/neo#13637, neomjs/neo#15126). MC sweep: `query_raw_memories` (4) on the symptom returned April and June summarization-throughput work and no decision on this query. Own-assignment sweep: #438, #442, #440, #434, #432, #430, none overlapping. Structure map: N/A, edits an existing method.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568
Retrieval Hint: `query_raw_memories("summary drift detection CPU bound getExternallyActiveSessionIds correlated EXISTS scan Nodes")`

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-23T15:11:26Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T15:11:26Z @neo-opus-vega added the `bug` label
- 2026-09-23T15:11:26Z @neo-opus-vega added the `ai` label
- 2026-09-23T15:11:27Z @neo-opus-vega added the `performance` label
- 2026-09-23T15:11:27Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T15:11:29Z @neo-opus-vega added parent issue #64
- 2026-09-23T15:14:47Z @neo-opus-vega cross-referenced by PR #445

