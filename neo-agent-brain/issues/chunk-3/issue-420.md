---
id: 420
title: Sync spec's task-state double records clocks the real writer never keeps
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T01:41:09Z'
updatedAt: '2026-09-23T02:27:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/420'
author: neo-opus-vega
commentsCount: 0
parentIssue: 194
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-23T02:27:20Z'
---
# Sync spec's task-state double records clocks the real writer never keeps

## Context

`TenantRepoSyncService.spec.mjs` drives the service against an in-memory task-state double — `createInMemoryTaskStateService`, spec `:94-102` at dev@1fc890c, 123 call sites in a 9,542-line spec. The double's `markCompleted` / `markSkipped` / `markFailed` record numeric `completedAt` / `skippedAt` / `failedAt`, and its `markStarted` a `startedAt`. The real `TaskStateService` (`createInitialTaskState`, `ai/daemons/orchestrator/services/TaskStateService.mjs:9`) has none of those fields: it writes `lastSuccessAt`, `lastErrorAt`, `lastExitCode`, `lastCompletion` and, since PR #418, `lastCompletionAt`.

The divergence has already cost a review round. PR #418's first head shipped `describeHolderYield` written against the double's clocks; on the real writer it returned `cycleAt: null` on every path, and the suite stayed green because the only task state it ever saw was the double's. @neo-gpt-emmy reproduced it in review 5285695130 and named the class: *"a test double with extra fields can make a disconnected reader look complete."*

## The Problem

Three arms use the phantom clocks as the disposition oracle — `:1434` (`skippedAt` truthy), `:8729-8730` (completed ⇒ `completedAt > 0`, `skippedAt` undefined) and `:8912-8913` (deferred ⇒ `skippedAt > 0`, `completedAt` undefined). Every other call site needs only `getTaskState().lastCompletion` and `running`. So the spec pins the completed-vs-skipped contract through fields production does not have, while the fields that carry the disposition in production — `lastExitCode` (`0` completed, `null` skipped, the code on failure), `lastSuccessAt`, `lastErrorAt`, `lastCompletionAt` — are never exercised by the service's largest consumer spec. A reader written against this spec's view of task state is written against fiction, and the suite cannot tell.

## The Architectural Reality

- The service calls exactly `getTaskState`, `markStarted`, `markCompleted`, `markSkipped`, `markFailed` (grep of `TenantRepoSyncService.mjs`: 2/1/1/2/3 sites). The real class has all five (`TaskStateService.mjs:262`, `:310`, `:341`, `:412`); nothing the double offers is missing from it.
- Sibling precedent in the tree: `pipeline.spec.mjs`'s production-composition arm and, since #418, `heavyMaintenanceStarvationWatchdog.spec.mjs` and `TaskStateService.spec.mjs` build the real writer over a temp state file — `Neo.create(TaskStateService, {stateFile, taskDefinitions, writeLogFn})` then `configure(…)`.
- The spec already owns a per-test temp dir (`tmpDir`, `beforeEach :257`, `afterEach :264`); the state file has a home.

## The Fix

1. Delete `createInMemoryTaskStateService`; the factory returns the real `TaskStateService` configured on `path.join(tmpDir, 'task-state.json')` with a `tenant-repo-sync` task definition. Keep the factory's name and call shape if that leaves the smaller diff — the 123 sites should not need to change.
2. Rewrite the three disposition arms against the real record: completed ⇒ `lastExitCode: 0`, `lastSuccessAt` set, `lastCompletionAt` set; skipped/deferred ⇒ `lastExitCode: null`, `lastSuccessAt` unchanged, `lastCompletionAt` set. The `lastCompletion` assertions stay as they are.
3. No production change. If a call site needs something the real writer lacks, that is a finding about the service and a separate ticket — not a reason to grow the double back.

## Decision Record impact

none. Test fidelity only; aligned with the real-writer-over-temp-file precedent above rather than with an ADR. Structure map (`npm run ai:structure-map -- --files --loc`): owning folder `ai/daemons/orchestrator/services`; no file is added or moved, placement N/A.

## Acceptance Criteria

- [ ] **AC-1** — `TenantRepoSyncService.spec.mjs` contains no task-state double: no `createInMemoryTaskStateService`, and no task-state read of a phantom clock — `grep -c 'getTaskState([^)]*)\.\(completedAt\|skippedAt\|failedAt\|startedAt\)\|taskState\[[^]]*\]\.\(completedAt\|skippedAt\|failedAt\|startedAt\)'` over the spec is 0. The service's own `details.skippedAt` and recovery-episode `failedAt` are different surfaces and stay. Every task state the spec drives is a real `TaskStateService` over a temp file.
- [ ] **AC-2** — The three disposition arms (`:1434`, `:8729-8730`, `:8912-8913` at dev@1fc890c) assert the real writer's fields and still discriminate: a mutation that turns the service's `markSkipped` call into `markCompleted` makes at least one of them red; receipt in the PR body.
- [ ] **AC-3** — The spec stays green where it already executes (`brain-unit.yml`'s executing list since #418) with the same test count and no test's subject weakened; local wall time before/after recorded in the PR body, since the real writer persists a file per mark.

## Out of Scope

- Any change to `TaskStateService` or `TenantRepoSyncService` production code.
- Other specs' task-state doubles — `grep "markCompleted\s*:"` over `test/playwright/unit` finds none outside this spec; a new one gets the same shape as a separate leaf.

## Avoided Traps

- ⛔ Do not repair the double by teaching it the real writer's fields — a hand-maintained mirror decays into this ticket again, and it is the shape #418's R1 was told not to take.
- ⛔ Do not weaken the disposition arms to `lastCompletion.status` alone — those arms exist because the task-state RECORD carries the disposition, which is what readers like the starvation receipt consume.

## Related

#415 / PR #418 (the reader that exposed it; review 5285695130) · #64 (the starvation-receipt lane) · #194 (epic: make the retained Brain test suite real — this is a leaf of that outcome; @neo-gpt-emmy owns the epic) · `TaskStateService.spec.mjs` (the writer contract).

Live latest-open sweep: checked latest 20 open Brain issues at 2026-09-23T01:38Z (created-desc) — none equivalent; nearest #201 (run the retained suite in CI — execution, not fidelity) and #415 (resolved by #418). A2A in-flight sweep: 30 most recent messages, all read-states, through 01:39Z — no claim on this scope. Memory Core sweep (`query_raw_memories`, task-state double / real writer nouns): no prior decision; the nearest records are tenant-sync behaviour lanes. Own-assignment sweep: #417, #415, #411, #237, #23, #64, #65 — none equivalent. `ask_knowledge_base` unavailable on the local plane until the corpus tenant is active (#411).

Origin Session ID: fc04c361-0cae-4a80-9506-fa2ef4785d2b
Retrieval Hint: "TenantRepoSyncService spec task-state double completedAt skippedAt real TaskStateService temp state file"


## Timeline

- 2026-09-23T01:41:09Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T01:41:11Z @neo-opus-vega added the `bug` label
- 2026-09-23T01:41:11Z @neo-opus-vega added the `ai` label
- 2026-09-23T01:41:11Z @neo-opus-vega added the `testing` label
- 2026-09-23T01:41:11Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T01:41:35Z @neo-opus-vega added parent issue #194
- 2026-09-23T01:58:51Z @neo-opus-vega cross-referenced by PR #421
- 2026-09-23T02:27:20Z @tobiu referenced in commit `5027afc` - "Merge pull request #421 from neomjs/vega/420-real-task-state-in-sync-spec

test(orchestrator): the sync spec drives the real task-state writer, not a double with clocks of its own (#420)"
- 2026-09-23T02:27:20Z @tobiu closed this issue
- 2026-09-23T02:29:44Z @neo-opus-vega cross-referenced by #64

