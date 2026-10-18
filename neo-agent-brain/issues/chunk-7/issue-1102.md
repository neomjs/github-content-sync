---
id: 1102
title: Convert 191 Orchestrator and daemon service B4 shared-config test writes
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - architecture
assignees:
  - neo-gemini-pro
createdAt: '2026-10-18T04:08:29Z'
updatedAt: '2026-10-18T18:17:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1102'
author: neo-gemini-pro
commentsCount: 1
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
closedAt: '2026-10-18T18:17:19Z'
---
# Convert 191 Orchestrator and daemon service B4 shared-config test writes

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T04:05:14Z` (`#1101`, `#1098`, `#1081`, `#1048`, `#1047`, `#1014`, `#1013`, `#1012`, `#1010`, `#1005`, `#979`, `#978`, `#977`, `#969`, `#957`, `#936`, `#933`, `#904`, `#875`, `#850`); no duplicate found (`#1098` is scoped to the `RemDigestion` + `github-workflow` `101`-write conversion).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T04:05:05Z`; no competing claim on converting the `6` `ai/daemons/orchestrator/` test files (`191` writes).
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"check-aiconfig-test-mutation B4 shared-config write concentrations Orchestrator DeploymentStateBridgeService configSnapshot"` at `2026-10-18T04:05:07Z`; surfaced `#541`, `#1086`, `#1090`, `#1092`, `#1091`, `#1096`, `#1097`, `#1099`, and `#1098`.
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` at `2026-10-18T04:05:02Z`; only `#1098` assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/daemons/orchestrator/Orchestrator.mjs`, `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs`, `ai/daemons/orchestrator/services/SwarmHeartbeatService.mjs`, `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs`, `ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## Context

`#541` (PR `#1086`), `#1090` (PR `#1092`), `#1091` (PR `#1099`), and `#1098` (PR `#1103`) converted `TextEmbeddingService` (`2` specs), `VectorService` (`7` specs), `GoldenPathSynthesizer` (`1` spec), `RemDigestion` (`2` specs), and `github-workflow` services (`6` specs) to isolated `configSnapshot` / `createConfigSnapshot()` injection backed by `ai/services/shared/configSnapshot.mjs`, while `#1096` (PR `#1097`) introduced the per-file ceiling ratchet (`ai/scripts/lint/shared-config-write-ceilings.json`), reducing global B4 shared-config writes from `655` to `342` across `37` remaining test files.

## The Problem

All `6` `ai/daemons/orchestrator/` unit test files in `ai/scripts/lint/shared-config-write-ceilings.json` still mutate shared `AiConfig` directly (`191` B4 shared-config writes total, including all `22` remaining `Object.assign` writes repo-wide):
- `test/playwright/unit/ai/daemons/orchestrator/services/DeploymentStateBridgeService.spec.mjs` (`66` writes)
- `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.spec.mjs` (`64` writes)
- `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.invariants.spec.mjs` (`33` writes)
- `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.cloudOnlyGate.spec.mjs` (`12` writes)
- `test/playwright/unit/ai/daemons/orchestrator/services/TenantRepoSyncService.spec.mjs` (`11` writes)
- `test/playwright/unit/ai/daemons/orchestrator/services/SwarmHeartbeatService.spec.mjs` (`5` writes)

## The Architectural Reality

- `ai/services/shared/configSnapshot.mjs` (`cloneConfigSubtree`, `validateConfigSnapshot`), `ai/services/shared/serviceProxy.mjs` (`makeSafe` `set` trap forwarding), and `ai/scripts/lint/shared-config-write-ceilings.json` provide the canonical ADR-0019 §4 B4 test-isolation seam and per-file ratchet.
- `Orchestrator`, `DeploymentStateBridgeService`, `SwarmHeartbeatService`, `TenantRepoSyncService`, and `ConfiguredTaskDefinitionsService` still read module-imported `AiConfig` directly across methods and helpers exercised by those `6` unit test suites.

## The Fix

1. Add `configSnapshot: null`, `beforeSetConfigSnapshot(value)` (using `validateConfigSnapshot` with non-cloning `#configSnapshot` getter/setter so in-place test overrides work cleanly), `createConfigSnapshot(overrides = {})` (using `cloneConfigSubtree`), and `#getConfig()` to `Orchestrator`, `DeploymentStateBridgeService`, `SwarmHeartbeatService`, and `TenantRepoSyncService`, and accept an optional `configSnapshot = null` parameter in `ConfiguredTaskDefinitionsService.buildConfiguredTaskDefinitions` and its task builder helpers.
2. Convert all `191` shared-config writes across the `6` `ai/daemons/orchestrator/` unit test files (`DeploymentStateBridgeService.spec.mjs`, `Orchestrator.spec.mjs`, `Orchestrator.invariants.spec.mjs`, `Orchestrator.cloudOnlyGate.spec.mjs`, `TenantRepoSyncService.spec.mjs`, `SwarmHeartbeatService.spec.mjs`) to inject isolated snapshots and assert `findSharedConfigMutations(source)` is `[]`.
3. Run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` to remove all `6` orchestrator test files from `ai/scripts/lint/shared-config-write-ceilings.json` (`37 -> 31` files, `342 -> 151` writes, `0` `Object.assign` writes remaining repo-wide), and track the remaining `151` writes across `31` files in `#1115`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Orchestrator.configSnapshot` / `Orchestrator.createConfigSnapshot(overrides)` (`ai/daemons/orchestrator/Orchestrator.mjs`) | ADR-0019 §4 B4 + `ai/services/shared/configSnapshot.mjs` | Optional plain-object snapshot overriding `AiConfig` reads; `null` reads live `AiConfig`; `createConfigSnapshot` clones `AiConfig` subtrees (`orchestrator`, `engines`, `memoryCore`, `session`, `debug`, `neoRootDir`, `projectRoot`, `hierarchyPath`, `graphProvider`, `embeddingProvider`, `modelProvider`, `data`) with overrides | Throws `TypeError` on invalid non-null `configSnapshot` via `validateConfigSnapshot` | JSDoc on config & methods | `Orchestrator.spec.mjs`, `Orchestrator.invariants.spec.mjs`, `Orchestrator.cloudOnlyGate.spec.mjs` |
| `DeploymentStateBridgeService.configSnapshot` / `DeploymentStateBridgeService.createConfigSnapshot(overrides)` (`ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs`) | ADR-0019 §4 B4 + `ai/services/shared/configSnapshot.mjs` | Optional plain-object snapshot overriding `AiConfig` reads; `null` reads live `AiConfig` | Throws `TypeError` on invalid non-null `configSnapshot` via `validateConfigSnapshot` | JSDoc on config & methods | `DeploymentStateBridgeService.spec.mjs` |
| `TenantRepoSyncService.configSnapshot` / `TenantRepoSyncService.createConfigSnapshot(overrides)` (`ai/daemons/orchestrator/services/TenantRepoSyncService.mjs`) | ADR-0019 §4 B4 + `ai/services/shared/configSnapshot.mjs` | Optional plain-object snapshot overriding `AiConfig` reads; `null` reads live `AiConfig` | Throws `TypeError` on invalid non-null `configSnapshot` via `validateConfigSnapshot` | JSDoc on config & methods | `TenantRepoSyncService.spec.mjs` |
| `SwarmHeartbeatService.configSnapshot` / `SwarmHeartbeatService.createConfigSnapshot(overrides)` (`ai/daemons/orchestrator/services/SwarmHeartbeatService.mjs`) | ADR-0019 §4 B4 + `ai/services/shared/configSnapshot.mjs` | Expanded `createConfigSnapshot` to include `orchestrator`, `session`, and `data` subtrees and routed `pulse()`, `getPulseIdentities()`, `getIdentityParticipation()`, `emitGitHubNotificationWakes()`, and path helpers through `#getConfig()` | Throws `TypeError` on invalid non-null `configSnapshot` via `validateConfigSnapshot` | JSDoc on config & methods | `SwarmHeartbeatService.spec.mjs` |
| `buildConfiguredTaskDefinitions(runtime, configSnapshot)` (`ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs`) | ADR-0019 §4 B4 | Reads `configSnapshot || AiConfig` across `buildConfiguredTaskDefinitions` and task helper builders | Defaults to `AiConfig` when `configSnapshot` is omitted/`null` | JSDoc on functions | `Orchestrator.spec.mjs`, `Orchestrator.invariants.spec.mjs` |
| `ai/scripts/lint/shared-config-write-ceilings.json` (`check-aiconfig-test-mutation.mjs`) | `#1096` / PR `#1097` | Ratchets down from `37` to `31` files (`342 -> 151` writes, `0` `Object.assign` writes remaining), dropping all `6` converted orchestrator spec entries | Fails CI on any write in removed/zero-ceiling files | Schema header in JSON | `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] AC-1: `Orchestrator`, `DeploymentStateBridgeService`, `TenantRepoSyncService`, `SwarmHeartbeatService`, and `ConfiguredTaskDefinitionsService` support isolated `configSnapshot` / `createConfigSnapshot(overrides)` injection backed by `ai/services/shared/configSnapshot.mjs` with zero `lint-config-template-ssot` or `check-aiconfig-antipatterns` violations.
- [ ] AC-2: All `191` B4 shared-config writes across `DeploymentStateBridgeService.spec.mjs` (`66`), `Orchestrator.spec.mjs` (`64`), `Orchestrator.invariants.spec.mjs` (`33`), `Orchestrator.cloudOnlyGate.spec.mjs` (`12`), `TenantRepoSyncService.spec.mjs` (`11`), and `SwarmHeartbeatService.spec.mjs` (`5`) are converted to `configSnapshot`, and each spec asserts `findSharedConfigMutations(source)` is `[]`.
- [ ] AC-3: `ai/scripts/lint/shared-config-write-ceilings.json` is tightened via `--tighten-ceilings`, removing all `6` converted orchestrator spec entries and reducing global B4 shared-config writes from `342` (`37` files) to `151` (`31` files, `0` `Object.assign` writes), with `#1115` tracking the remaining `31` files.

## Out of Scope

- Converting the remaining `151` B4 shared-config writes across the final `31` service/MCP test files (tracked in `#1115`).

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`, `#1090`, `#1091`, and `#1098`; tests must inject an isolated `configSnapshot` rather than proliferating per-leaf function hooks.

## Related

- Succeeds: `#541` / PR `#1086`, `#1090` / PR `#1092`, `#1091` / PR `#1099`, `#1098` / PR `#1103`
- Succeeded by: `#1115`
- Related: `#508`, `#1096` / PR `#1097`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "findSharedConfigMutations B4 191 shared-config writes 6 orchestrator test files Orchestrator DeploymentStateBridgeService configSnapshot"


## Timeline

- 2026-10-18T04:08:30Z @neo-gemini-pro added the `enhancement` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `ai` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `testing` label
- 2026-10-18T04:08:30Z @neo-gemini-pro added the `architecture` label
- 2026-10-18T04:11:52Z @neo-gemini-pro cross-referenced by PR #1103
- 2026-10-18T04:43:44Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-18T05:53:46Z @neo-gemini-pro cross-referenced by #1115
- 2026-10-18T05:54:29Z @neo-gemini-pro changed title from **Convert remaining Orchestrator, DeploymentStateBridgeService, and service B4 shared-config writes** to **Convert 191 Orchestrator and daemon service B4 shared-config test writes**
- 2026-10-18T06:05:13Z @neo-gemini-pro cross-referenced by PR #1116
- 2026-10-18T06:30:28Z @neo-gemini-pro referenced in commit `834119c` - "fix(ai): preserve prototype-call and readiness-coordinate compatibility (#1102)"
- 2026-10-18T07:31:20Z @neo-gemini-pro referenced in commit `28f4daf` - "fix(ai): deep-clone orchestrator configSnapshots and remove singleton fallbacks (#1102)"
- 2026-10-18T07:48:53Z @neo-gemini-pro referenced in commit `65d46fb` - "fix(ai): preserve resolveCloudOnlyEnabled anchor and stub getGraphDb in SwarmHeartbeatService.spec (#1102)"
- 2026-10-18T09:03:04Z @neo-gemini-pro cross-referenced by PR #1135
- 2026-10-18T17:51:46Z @neo-gemini-pro referenced in commit `e5ed7d2` - "feat(ai): convert Orchestrator and daemon service B4 test writes (#1102)"
- 2026-10-18T17:51:46Z @neo-gemini-pro referenced in commit `daacdd2` - "fix(ai): preserve prototype-call and readiness-coordinate compatibility (#1102)"
- 2026-10-18T17:51:46Z @neo-gemini-pro referenced in commit `69aa747` - "fix(ai): deep-clone orchestrator configSnapshots and remove singleton fallbacks (#1102)"
- 2026-10-18T17:51:46Z @neo-gemini-pro referenced in commit `41fa166` - "fix(ai): preserve resolveCloudOnlyEnabled anchor and stub getGraphDb in SwarmHeartbeatService.spec (#1102)"
### @neo-gemini-pro - 2026-10-18T18:17:18Z

Closing as not planned per @tobiu's architectural review: `configSnapshot` violates ADR-0019 (`AiConfig` is a reactive hierarchical `Neo.state.Provider` SSOT). Follow-up tickets will revert/remove `configSnapshot` across `ai/` and `test/`.

- 2026-10-18T18:17:19Z @neo-gemini-pro closed this issue

