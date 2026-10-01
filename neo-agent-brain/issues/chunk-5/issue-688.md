---
id: 688
title: 'IssueIngestor.spec restores a pre-init null GraphService.db, breaking the next graph spec in its worker'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T13:38:46Z'
updatedAt: '2026-10-01T13:55:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/688'
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
closedAt: '2026-10-01T13:55:28Z'
---
# IssueIngestor.spec restores a pre-init null GraphService.db, breaking the next graph spec in its worker

## Context

Brain Unit failed on PR #681 (run 36868231160) with one head-only failure: `FileSystemIngestor.spec.mjs` › "should dynamically ignore high-noise path patterns while preserving structural mapping" → `TypeError: Cannot read properties of null (reading 'getAdjacentNodes')`. #681 touches neither spec nor the memory-core ingestion code. It only changes how the CI workers are filled, which is the class #667 fixed for two other specs.

## The Problem

Measured on `dev` (Brain 8534e18) with the head report's worker assignment:

- `FileSystemIngestor.spec.mjs` passes alone (6/6).
- On CI worker 120 it ran after 66 other specs. Replayed in that order on one worker, it fails with the CI error. The replay leaves out the one case that failed on other workers in CI too, because a failure restarts the worker.
- Bisecting that chain leaves one spec. `IssueIngestor.spec.mjs` followed by `FileSystemIngestor.spec.mjs` on one worker fails the same way: 1 failed, 24 passed.

## The Architectural Reality

`IssueIngestor.spec.mjs` (first `describe`):
- `beforeAll` imports `ai/services.mjs` and captures `GraphService.db` straight away (`_originalGraphDb`). It then swaps in a stub `{nodes, edges, getAdjacentNodes, addNode, updateNode}`.
- `afterAll` writes the captured value back.

`FileSystemIngestor.spec.mjs` `beforeAll` re-initialises only when `SystemLifecycleService._initPromise` is unset. Otherwise it awaits `ready()` and uses `GraphService.db` as it finds it.

The capture runs before the GraphService singleton's own initialisation has finished, so the value captured and later "restored" is `null`. The worker is left with an initialised lifecycle and no graph. Sibling specs that swap the graph wait first (`DatabaseService.graphReplaceReadAtPreserved.spec.mjs`, the memory-core `Server.spec.mjs`: `await GraphService.ready()`).

## The Fix

`IssueIngestor.spec.mjs` awaits `GraphService.ready()` before it captures `GraphService.db`, so `afterAll` hands back the real graph. Test-only, one spec.

## Acceptance Criteria

- [ ] AC-1: `IssueIngestor.spec.mjs` followed by `FileSystemIngestor.spec.mjs` on one worker passes. It fails without the fix.
- [ ] AC-2: `IssueIngestor.spec.mjs` passes alone, as before.

## Out of Scope

- Making `FileSystemIngestor.spec.mjs` re-initialise defensively. The defect is the state the earlier spec leaves, not the reader that trusts it.
- Other specs that swap service singletons without waiting.

## Related

#667 / #668 (the same class: worker order decides pass/fail and the head-vs-base check blames the PR that reshuffles) · #681 (where it surfaced)

Live latest-open sweep: the latest open Brain issues at 2026-10-01T13:45Z (#687 down), no equivalent; org search for "IssueIngestor spec" and "FileSystemIngestor getAdjacentNodes": closed, unrelated hits only. A2A in-flight sweep (last 60 min): no claim. Own-assignment sweep: none on this surface.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a
Retrieval Hint: `query_raw_memories("IssueIngestor spec restores null GraphService.db worker order FileSystemIngestor getAdjacentNodes")`

## Timeline

- 2026-10-01T13:38:47Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T13:38:48Z @neo-opus-ada added the `bug` label
- 2026-10-01T13:38:48Z @neo-opus-ada added the `ai` label
- 2026-10-01T13:38:48Z @neo-opus-ada added the `testing` label
- 2026-10-01T13:40:11Z @neo-opus-ada cross-referenced by PR #689
- 2026-10-01T13:40:50Z @neo-opus-ada cross-referenced by PR #681
- 2026-10-01T13:55:28Z @tobiu referenced in commit `741f9f3` - "test(ingestion): IssueIngestor.spec captures the graph after GraphService is ready (#688) (#689)

The spec captured GraphService.db right after importing the services, before
the singleton had its graph, so afterAll restored null. The next graph spec in
the worker then found an initialised lifecycle with no graph:
FileSystemIngestor.spec failed with null.getAdjacentNodes on PR #681's CI
worker 120. It now awaits GraphService.ready() before the capture, as its
siblings do."
- 2026-10-01T13:55:28Z @tobiu closed this issue

