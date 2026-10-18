---
id: 1102
title: 'Convert remaining Orchestrator, DeploymentStateBridgeService, and service B4 shared-config writes'
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - architecture
assignees: []
createdAt: '2026-10-18T04:08:29Z'
updatedAt: '2026-10-18T04:08:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1102'
author: neo-gemini-pro
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
# Convert remaining Orchestrator, DeploymentStateBridgeService, and service B4 shared-config writes

unowned-rationale: Successor to #1098 (#1091 / #1090 / #541) for the remaining 342 B4 shared-config writes across 37 test files after the RemDigestion and github-workflow conversion; parked with measured file and leaf rankings for the next maintainer pickup.

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T04:05:14Z` (`#1101`, `#1098`, `#1081`, `#1048`, `#1047`, `#1014`, `#1013`, `#1012`, `#1010`, `#1005`, `#979`, `#978`, `#977`, `#969`, `#957`, `#936`, `#933`, `#904`, `#875`, `#850`); no duplicate found (`#1098` is scoped to the `RemDigestion` + `github-workflow` `101`-write conversion).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T04:05:05Z`; no competing claim on converting the remaining `37` daemon/service test files.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"check-aiconfig-test-mutation B4 shared-config write concentrations Orchestrator DeploymentStateBridgeService configSnapshot"` at `2026-10-18T04:05:07Z`; surfaced `#541`, `#1086`, `#1090`, `#1092`, `#1091`, `#1096`, `#1097`, `#1099`, and `#1098`.
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` at `2026-10-18T04:05:02Z`; only `#1098` assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/daemons/orchestrator/Orchestrator.mjs`, `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## Context

`#541` (PR `#1086`), `#1090` (PR `#1092`), `#1091` (PR `#1099`), and `#1098` converted `TextEmbeddingService` (`2` specs), `VectorService` (`7` specs), `GoldenPathSynthesizer` (`1` spec), `RemDigestion` (`2` specs), and `github-workflow` services (`6` specs) to isolated `configSnapshot` / `createConfigSnapshot()` injection backed by `ai/services/shared/configSnapshot.mjs`, while `#1096` (PR `#1097`) introduced the per-file ceiling ratchet (`ai/scripts/lint/shared-config-write-ceilings.json`), reducing global B4 shared-config writes from `655` to `342` across `37` remaining test files.

## The Problem

`node ai/scripts/lint/check-aiconfig-test-mutation.mjs` still reports `342 shared-config write(s) across 104 leaf/leaves (320 assignment, 22 Object.assign)` across `37` remaining test files in `ai/scripts/lint/shared-config-write-ceilings.json`:
- Top remaining test file concentrations:
  - `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.spec.mjs` (`65` writes)
  - `test/playwright/unit/ai/daemons/orchestrator/services/DeploymentStateBridgeService.spec.mjs` (`57` writes)
  - `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.invariants.spec.mjs` (`45` writes)
  - `test/playwright/unit/ai/daemons/orchestrator/services/TenantRepoSyncService.spec.mjs` (`13` writes)
  - `test/playwright/unit/ai/services/memory-core/SessionSummarization.spec.mjs` (`13` writes)
  - `test/playwright/unit/ai/mcp/server/shared/services/DestructiveOperationGuard.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/knowledge-base/source/ApiSource.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/rem-observability.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/knowledge-base/source/SkillSource.spec.mjs` (`10` writes)
  - `test/playwright/unit/ai/mcp/server/github-workflow/ConfigCompleteness.spec.mjs` (`8` writes)
- Top remaining leaf targets:
  - `.orchestrator.deploymentMode` (`17`), `AiConfig.orchestrator.deploymentStateBridge` (`15`), `.data.debug` (`15`), `.neoRootDir` (`12`), `.orchestrator.chroma` (`11`), `.hierarchyPath` (`11`), `.graphProvider` (`9`), `.embeddingProvider` (`9`), `.orchestrator.lms` (`7`), `.modelProvider` (`7`).

## The Architectural Reality

- `ai/services/shared/configSnapshot.mjs` (`cloneConfigSubtree`, `validateConfigSnapshot`), `ai/services/shared/serviceProxy.mjs` (`makeSafe` `set` trap forwarding), and `ai/scripts/lint/shared-config-write-ceilings.json` provide the canonical ADR-0019 §4 B4 test-isolation seam and per-file ratchet.
- `Orchestrator`, `DeploymentStateBridgeService`, `TenantRepoSyncService`, `SessionService`, `DestructiveOperationGuard`, and Knowledge Base source services still read module-imported `aiConfig` directly in unit tests that mutate shared config leaves.

## The Fix

1. Extend the `configSnapshot` / `beforeSetConfigSnapshot` / `createConfigSnapshot()` pattern (using `ai/services/shared/configSnapshot.mjs`) to the remaining target daemons and services (`Orchestrator`, `DeploymentStateBridgeService`, `TenantRepoSyncService`, `SessionService`, `DestructiveOperationGuard`, KB sources, etc.).
2. Convert the remaining `37` test files (`342` shared-config writes) to inject isolated snapshots and run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` to drop their entries from `ai/scripts/lint/shared-config-write-ceilings.json`.
3. Once all test files reach `0` shared-config writes, retire `ai/scripts/lint/shared-config-write-ceilings.json` so any shared-config write in `test/**` fails the lint gate unconditionally.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `findSharedConfigMutations` (`ai/scripts/lint/check-aiconfig-test-mutation.mjs`) | ADR-0019 §4 B4 + `#508` + `#1096` | Per-file shared-config write counts ratchet down to `0` across the remaining `37` files (`342` writes) | Per-file ceiling ratchet (`#1096`) blocks regressions on clean/ratcheted files | JSDoc in `check-aiconfig-test-mutation.mjs` | `npm run test-unit` + `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] Convert the remaining `37` test files (`342` shared-config writes) across `Orchestrator`, `DeploymentStateBridgeService`, `TenantRepoSyncService`, `SessionService`, `DestructiveOperationGuard`, KB sources, and remaining specs to isolated `configSnapshot` injection with no per-leaf `*Fn_` seams.
- [ ] Run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` and retire the non-zero ceiling table once `0` shared-config writes remain across `test/**`.

## Out of Scope

- `TextEmbeddingService`, `VectorService`, `GoldenPathSynthesizer`, `RemDigestion`, and `github-workflow` syncer/service specs completed in `#541` (`#1086`), `#1090` (`#1092`), `#1091` (`#1099`), and `#1098`.

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`, `#1090`, `#1091`, and `#1098`; tests must inject an isolated `configSnapshot` rather than proliferating per-leaf function hooks.

## Related

- Succeeds: `#541` / PR `#1086`, `#1090` / PR `#1092`, `#1091` / PR `#1099`, `#1098`
- Related: `#508`, `#1096` / PR `#1097`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "findSharedConfigMutations B4 342 shared-config writes 37 test files Orchestrator DeploymentStateBridgeService"


## Timeline

- 2026-10-18T04:08:30Z @neo-gemini-pro added the `enhancement` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `ai` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `testing` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `architecture` label
- 2026-10-18T04:11:52Z @neo-gemini-pro cross-referenced by PR #1103

