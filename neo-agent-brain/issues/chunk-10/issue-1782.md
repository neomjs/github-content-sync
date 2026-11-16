---
id: 1782
title: Failed graph rollback duplicates edges in adjacency reads
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-16T02:58:11Z'
updatedAt: '2026-11-16T08:09:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1782'
author: neo-gpt-sophie
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
closedAt: '2026-11-16T08:09:47Z'
---
# Failed graph rollback duplicates edges in adjacency reads

Please comment to claim before starting; check current assignees and linked PRs.

## Context

An ordinary failed graph transaction can leave duplicate secondary-index references even though persistence rolled back correctly. This was independently reproduced while validating #1510, on base `1f884fed59406b481e22346ee9ca68731a59c354` and the feature branch. The base Database blob is `c32bdbc0c8678f5df9367863b46f649edd90c30b`. Probes on Node 24.22.1 and 25.9.0 agree; the repository declares Node >=24 and pins Engine commit `cc24c59e0d2470b75cfccaaa62b52cfd053bddb2`.

## The Problem

Using the actual `Neo.ai.graph.Database` with initialized `SQLite({dbPath: ':memory:'})`:

```js
graph.transaction(() => {
    graph.addNode({id: 'a', label: 'ORDINARY'});
    graph.addNode({id: 'b', label: 'ORDINARY'});
    graph.addEdge({id: 'e', source: 'a', target: 'b', type: 'RELATED_TO'});
});
try {
    graph.transaction(() => {
        graph.removeEdge('e');
        throw new Error('abort');
    });
} catch {}

graph.edges.getByIndex('source', 'a').map(edge => edge.id); // ['e', 'e']
graph.edges.getCount();                                  // 1
graph.getAdjacentNodes('a').map(node => node.id);           // ['b', 'b']
```

Before the failure the source index contains one `e`. Afterward SQLite rows, GraphLog and `lastSyncId` are unchanged. This is duplicate adjacency, not duplicate persistence. No live store is needed to reproduce it.

## The Architectural Reality

`ai/graph/Database.mjs#transaction` records Store mutations and calls `rollbackTransaction` on ordinary errors. `ai/graph/Store.mjs` owns the source/target index references used by adjacency readers.

Design authority: the existing `Database.spec.mjs` rollback contract, “should instantly rollback Memory Collections if transaction boundary throws.” The same file already covers remove-by-key node/edge rollback, but its count assertions miss this index mismatch.

Prescription checked: the existing graph Database/Store boundary owns rollback and indexed identity. Inspect both the raw and hydrated references before choosing the exact repair; do not deduplicate output in each consumer.

## The Fix / Contract Ledger

| Surface | Required behavior | Refusal / edge | Evidence |
| --- | --- | --- | --- |
| Ordinary rollback | Restore exactly one source and target index reference per retained edge, consistent with the Store map | Repeated aborted removals and raw/hydrated records must not accumulate aliases | Actual Database + SQLite fixture |
| Adjacency reads | Return the original adjacency after rollback | Empty/no-match and unavailable-index semantics remain unchanged | `getAdjacentNodes` and both index lookups |
| Persistence/coherence | Preserve SQLite rows, GraphLog and cursor when the logical closure fails | Do not acknowledge an unrelated concurrent writer | Before/after physical and cursor assertions |

## Acceptance Criteria

- [ ] Add a regression reproducing the duplicate index/adjacency above before the repair.
- [ ] After one and repeated failed transactions, both source and target indexes contain one reference per surviving edge and adjacency matches the pre-transaction result. Cover raw and hydrated cache records.
- [ ] SQLite rows, GraphLog and `lastSyncId` remain unchanged for the failed closure; successful ordinary transactions still persist.
- [ ] Existing remove-by-key rollback and Store index suites pass using disposable fixtures.

Use the existing fixture in `test/playwright/unit/ai/graph/Database.spec.mjs` and the adjacent Store/SQLite specs. Run through `npm run test-unit -- <spec-path> --project=unit-brain --no-deps --workers=1`; these tests need no live Chroma, service restart or private credentials.

## Out of Scope / Avoided Traps

No live graph repair, new persistence schema, per-consumer deduplication, global cache flush, or change to protected-family admission. #1510 owns protected-write refusal and publication commit/cache behavior; its branch deliberately leaves this ordinary rollback defect unchanged. Coordinate the shared Database file at intake.

Related: #1510. Brain `#20` concerns global log acknowledgement and `#517` uncached deletion. Engine [#11595](https://github.com/neomjs/neo/issues/11595) fixed string-shaped removal payloads; [#16965](https://github.com/neomjs/neo/issues/16965) concerns missing indexes, not duplicated references.

Decision Record impact: none; restores the existing rollback and index contract.
unowned-rationale: Ready for a contributor; maintainers provide planning and review. The reproducer and verification use public source and disposable storage.
Origin Session ID: ac8ad587-00f3-460f-809f-4af3b8b265c9
Retrieval Hint: ordinary graph rollback duplicate edge source target indexes adjacency

Live latest-open + closed sweep: latest 20 open issues and closed `rollback index` search checked at 2026-11-16 02:58:10 UTC; 161 open (red). Earlier complete open-queue/symbol searches and the scoped closed records above found different failure classes, not this duplicate-reference defect. No owning ticket was found, so this independently reproduced bug is admitted separately.
MC sweep: `graph rollback duplicate edges adjacency index`, 3 results; prior protected-cache work and unrelated failures, no replacement for this ordinary-rollback outcome.
Own-assignment sweep: 9 open; #1510/#1296 are protected-family source work, not this ordinary rollback repair. Recent all-state A2A claims checked; no overlapping claim found. Structure map passed; existing `ai/graph/Database.mjs` / `Store.mjs` own the boundary; no new directory prescribed.


## Timeline

- 2026-11-16T02:58:13Z @neo-gpt-sophie added the `bug` label
- 2026-11-16T02:58:13Z @neo-gpt-sophie added the `good first issue` label
- 2026-11-16T02:58:13Z @neo-gpt-sophie added the `help wanted` label
- 2026-11-16T02:58:13Z @neo-gpt-sophie added the `ai` label
- 2026-11-16T02:58:14Z @neo-gpt-sophie added the `testing` label
- 2026-11-16T02:58:14Z @neo-gpt-sophie added the `agent-os` label
### @enidchen0125 - 2026-11-16T03:04:02Z

I'd like to claim this issue. I'll reproduce the rollback/index mismatch with the existing disposable Database/SQLite fixture, add raw and hydrated-record regressions, and keep the fix at the Store/Database boundary without changing #1510's protected-write behavior. Implementation and verification will use OpenAI Codex assistance, disclosed in the PR. I'll wait for assignment before starting.


### @github-actions - 2026-11-16T03:04:11Z

Thanks for claiming this, @enidchen0125, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@enidchen0125 -->

- 2026-11-16T03:21:38Z @enidchen0125 cross-referenced by PR #1785
- 2026-11-16T03:50:31Z @neo-gpt-sophie cross-referenced by PR #1786
- 2026-11-16T08:09:47Z @tobiu referenced in commit `41a6a75` - "fix(graph): keep secondary indexes in sync with hydration (#1782) (#1785)

Co-authored-by: enidchen0125 <196524738+enidchen0125@users.noreply.github.com>"
- 2026-11-16T08:09:48Z @tobiu closed this issue

