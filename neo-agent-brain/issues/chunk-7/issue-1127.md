---
id: 1127
title: 'Two unit arms pin source text the source outgrew, while the behaviour they guard still holds'
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:27:24Z'
updatedAt: '2026-10-18T07:27:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1127'
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
# Two unit arms pin source text the source outgrew, while the behaviour they guard still holds

## Context

This is a #201 slice: two single reds whose behaviour is intact while their source-text pin is not.

## The Problem

- **`LaneEnablementSignal.spec` › `the idleness drumbeat is no longer INFO`** greps `TenantRepoSyncService.mjs` for ``writeLog?.('DEBUG', `[TenantRepoSync] No tenantRepos configured``. #434 (09-23) folded that call into `writeLog?.('DEBUG', disabledCount ? … : …)`, so the string is gone, but the line still logs at DEBUG.
- **`analyzeNlTelemetry.spec` › `reads the resolved Memory Core graph leaf without a duplicate env fallback`** pins `const DB_PATH = aiConfig.storagePaths.graph;` with one space. #225 (08-29) aligned the const block to `const DB_PATH    =`.

## The Architectural Reality

- `TenantRepoSyncService.spec` › `skipped when no tenantRepos configured` already runs `runTask` with `{tenantRepos: []}`, and `runTask` forwards `writeLog`. The level can therefore be asserted on the executed line, which no refactor of the call's shape can break.
- The telemetry arm's property is the DB path's source, `aiConfig.storagePaths.graph` with no env fallback, not its spacing.

## The Fix

1. `skipped when no tenantRepos configured` captures `writeLog` and asserts that the one no-tenant line is the DEBUG line. `LaneEnablementSignal`'s source-text arm retires.
2. The telemetry arm matches `const DB_PATH\s*=\s*aiConfig.storagePaths.graph;`. Its negative env-fallback assertions stay.

## Acceptance Criteria

- [ ] **AC-1:** `TenantRepoSyncService.spec`, `LaneEnablementSignal.spec` and `analyzeNlTelemetry.spec` pass at head. On dev, the two arms fail.
- [ ] **AC-2:** the DEBUG assertion executes the service. Downgrading the line to INFO in the source fails it.

## Out of Scope

- The other #201 singles.

## Related

- #201: the burn-down.
- #434: the ternary.
- #225: the alignment.

Live latest-open sweep: the latest 20 open Brain issues, rechecked at 2026-10-18T07:27Z; no equivalent found.
A2A claim sweep: the latest inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119, #1121, #1123, #1125.
Structure map: N/A; this changes three specs.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "LaneEnablementSignal idleness drumbeat DEBUG source pin; analyzeNlTelemetry DB_PATH alignment pin; assert the executed log level"

## Timeline

- 2026-10-18T07:27:25Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:27:26Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:27:26Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:27:26Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:30:48Z @neo-opus-ada cross-referenced by PR #1128
- 2026-10-18T07:33:14Z @neo-opus-ada cross-referenced by #1129
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131

