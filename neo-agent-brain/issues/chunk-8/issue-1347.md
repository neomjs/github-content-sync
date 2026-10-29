---
id: 1347
title: The graph-backup completeness arm plants nodes that SQLite now refuses
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T13:42:22Z'
updatedAt: '2026-10-29T13:42:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1347'
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
---
# The graph-backup completeness arm plants nodes that SQLite now refuses

## Context

#201's retained suite has three reds left (comment 6482850355). One is `test/playwright/unit/ai/services/memory-core/DatabaseService.graphBackup.spec.mjs` › "fails loud when graph rows or source counts cannot be read (#16407)". It is engine-independent: it fails at both the 09-25 pin and `cc24c59e0d`.

## The Problem

The arm plants two `Nodes` rows with malformed JSON (`'{invalid-source-json'`, `'{invalid-target-json'`) and one malformed `Edges` row. It then expects the graph export to fail loud with `exported 0/3` and three `skippedIds`.

#555 (2026-09-26) added expression indexes over `Nodes`, among them `idx_nodes_label ON Nodes(json_extract(data, '$.label'))` at `ai/graph/storage/SQLite.mjs:113`. SQLite evaluates `json_extract` on every insert, so the plant itself throws `SqliteError: malformed JSON` (measured 2026-10-29 on `dev@2ed12394`) and the child never reaches the export. The arm has been red at its own setup since 09-26.

A running store can't hold the malformed node the arm plants. `INSERT` and `UPDATE` refuse it, and `CREATE INDEX` fails on a store that already holds one. `Edges` carry no such index, so an unreadable edge is still a reachable state, and the export's edge `catch` is still the path worth proving.

## The Architectural Reality

- The graph export in `ai/services/memory-core/DatabaseService.mjs` runs the node and edge loops in one shape. Each `JSON.parse` sits in a `try`, a failure counts into `skipped`/`skippedIds`, and `classifyExportCompleteness` then throws `PARTIAL_COLLECTION_EXPORT`.
- The node loop's `catch` only reports, so it stays. Removing it would change production code on #555's surface for no behavioural gain.

## The Fix

The arm plants valid source/target nodes and one malformed edge. It expects `exported 2/3`, `skipped 1` and `skippedIds ['edge:graph-backup-corrupt-edge']`. In the same child it asserts that a malformed `Nodes` insert is refused with `malformed JSON`. That assertion pins why only an edge is planted: if the index ever goes, the arm flips, and the node half needs its plant back. The count-read half (`GRAPH_COUNT_QUERY_FAILED`) is unchanged.

## Acceptance Criteria

- [ ] AC-1: the arm reaches the export and passes, failing loud on the unreadable edge with `exported 2/3`.
- [ ] AC-2: the arm asserts that `Nodes` refuses a malformed-JSON insert.
- [ ] AC-3: the count-read assertions are unchanged and pass.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds, with this arm among the fixed.

## Out of Scope

- Removing the node loop's `catch`.
- `lintGuides` (a custody decision) and `GuideToolParity` (#1346), #201's other reds.

## Avoided Traps

- Planting the malformed node through a store built without #555's indexes. That tests a state no running plane can hold.
- Dropping the arm. Its edge half still guards a reachable loss.

## Related

Parent: #201. Index: #555 (from #552). Original guard: `neomjs/neo#16407`.

Live latest-open + closed sweep: latest 20 open Brain issues + a closed search (graphBackup, "malformed JSON", 16407) at 2026-10-29T13:42:05Z, with no equivalent. A2A claim sweep: the unread queue at 13:41Z plus the latest 30 at 13:21Z, with no claim. Own assignments: #201 (parent), #1346 (sibling). Structure map: N/A, since this changes one spec.

Retrieval Hint: "graphBackup completeness arm malformed JSON expression index json_extract Nodes plant edge only"

## Timeline

- 2026-10-29T13:42:22Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T13:42:23Z @neo-opus-ada added the `bug` label
- 2026-10-29T13:42:23Z @neo-opus-ada added the `ai` label
- 2026-10-29T13:42:24Z @neo-opus-ada added the `testing` label
- 2026-10-29T13:44:26Z @neo-opus-ada cross-referenced by PR #1348
- 2026-10-29T13:57:10Z @neo-opus-ada cross-referenced by #1353

