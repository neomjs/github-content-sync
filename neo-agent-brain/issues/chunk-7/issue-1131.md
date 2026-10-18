---
id: 1131
title: Orchestrator.spec's task-envelope census lacks community-reconciliation
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:58:37Z'
updatedAt: '2026-10-18T07:58:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1131'
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
# Orchestrator.spec's task-envelope census lacks community-reconciliation

## Context

This is a #201 single: `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.spec.mjs` › `creates an isolated persisted-state envelope per task` fails on its key list, which received one extra entry, `community-reconciliation`.

## The Problem

`createInitialTaskState` keys the state by `Object.keys(buildTaskDefinitions(…))`, so the arm's pinned list is a census of the orchestrator's tasks. #104 (09-20) added `community-reconciliation` while the Brain unit suite did not run in CI, and the census was never updated. Today the definitions hold 25 tasks against the census's 24. The only difference is `community-reconciliation`, which sits between `tenant-repo-sync` and `dream`.

## The Architectural Reality

The census is worth keeping rather than deriving. Now that compareUnitFailures fails a PR that introduces a red, the next added task must update the census in its own PR, so every task addition is a conscious act.

## The Fix

Add `community-reconciliation` to the census after `tenant-repo-sync`.

## Acceptance Criteria

- [ ] **AC-1:** `Orchestrator.spec.mjs` passes at head. On dev, this arm fails.

## Out of Scope

- The other #201 singles.

## Related

- #201: the burn-down.
- #104: the task.

Live latest-open sweep: the latest 20 open Brain issues, rechecked at 2026-10-18T08:00Z; no equivalent found.
A2A claim sweep: the latest inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119, #1121, #1123, #1125, #1127, #1129.
Structure map: N/A; this changes one spec line.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "Orchestrator.spec task census community-reconciliation createInitialTaskState"

## Timeline

- 2026-10-18T07:58:37Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:58:38Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:58:38Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:58:39Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:59:32Z @neo-opus-ada cross-referenced by PR #1132

