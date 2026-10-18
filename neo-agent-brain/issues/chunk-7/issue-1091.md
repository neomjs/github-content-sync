---
id: 1091
title: Convert VectorService and GoldenPathSynthesizer B4 shared-config test writes
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - testing
  - architecture
assignees:
  - neo-gemini-pro
createdAt: '2026-10-18T00:04:06Z'
updatedAt: '2026-10-18T03:49:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1091'
author: neo-gemini-pro
commentsCount: 0
parentIssue: null
subIssues:
  - '[x] 1096 Shared-config writes in a test file may not exceed its ceiling'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-18T03:49:55Z'
---
# Convert VectorService and GoldenPathSynthesizer B4 shared-config test writes

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T00:03:20Z`; no duplicate found (`#1090` closed the `TextEmbeddingService` and `VectorService.batchFailureIsolation` slice; `#1096` adds the per-file ceiling ratchet; `#1098` tracks the remaining `443` writes across `45` test files).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T00:03:25Z`; no competing `[lane-claim]` or `[lane-intent]` on `VectorService` or `GoldenPathSynthesizer` B4 shared-config write isolation.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"findSharedConfigMutations shared-config write B4 Orchestrator DeploymentStateBridgeService"`; surfaced `#541`, `#1086`, and `#1090`.
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me`; `#1091` is self-assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/services/knowledge-base/VectorService.mjs`, `ai/services/graph/GoldenPathSynthesizer.mjs`, `ai/services/shared/configSnapshot.mjs`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## Context

`#541` (PR `#1086`) and `#1090` (PR `#1092`) established the shared `configSnapshot` / `beforeSetConfigSnapshot` / `createConfigSnapshot()` pattern in `ai/services/shared/configSnapshot.mjs` and converted `TextEmbeddingService` (`2` specs) plus `VectorService.batchFailureIsolation.spec.mjs` (`1` spec).

This ticket delivers the next bounded B4 conversion slice: the remaining `7` `VectorService` unit specs (`58` shared-config writes) and `GoldenPathSynthesizer.spec.mjs` (`44` shared-config writes), eliminating `102` shared-config writes across `8` test files (`545 -> 443` writes across `125` leaves: `421` assignment, `22` `Object.assign`, and `53 -> 45` files in `ai/scripts/lint/shared-config-write-ceilings.json`). Sibling `#1096` (PR `#1097`) added the per-file ceiling ratchet, and successor `#1098` tracks the remaining `443` writes across `45` test files.

## The Problem

1. **Remaining `VectorService` unit specs (`58` writes across `7` files):**
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.WorkVolumeBranching.spec.mjs` (`15` writes, plus `7` `KB_Config.data.*` writes on `VectorService` leaves)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.undeliverableGeometry.spec.mjs` (`11` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.persistenceNonConvergence.spec.mjs` (`9` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.tenantStamping.spec.mjs` (`9` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.leaseYield.spec.mjs` (`6` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.failureCarry.spec.mjs` (`4` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.ollamaIngestSeam.spec.mjs` (`4` writes)
   - `VectorService.mjs` still reads `aiConfig` / cross-server `mcConfig` directly in `#resolveLogPath()`, `#getHierarchyPath()`, `resolveTenantStamp()`, `getTenantIsolationConfig()`, `resolveEmbeddingGuardrail()`, `resolveEmbeddingPoisonGeneration()`, `generateIsolationEmbeddings()`, `embed()`, and `embedChunks()` instead of `#getConfig()`.
2. **`GoldenPathSynthesizer.spec.mjs` (`44` writes in `1` file):**
   - Mutates `aiConfig` leaves (`goldenPath*`, `graphProvider`, `embeddingProvider`, `embeddingModel`, `vectorDimension`, `ollama`, `openAiCompatible`, `orchestrator.corpusProjection`, `handoffFilePath`, `projectRoot`) directly across `GoldenPathSynthesizer` instance and static methods.

## The Architectural Reality

- `ai/services/shared/configSnapshot.mjs` (`cloneConfigSubtree`, `validateConfigSnapshot`) and `ai/services/shared/serviceProxy.mjs` (`makeSafe` `set` trap forwarding) landed in `#1090` (PR `#1092`).
- `GoldenPathSynthesizer` is a singleton (`Neo.setupClass(GoldenPathSynthesizer)`) whose unit tests invoke both singleton instance methods (`GoldenPathSynthesizer.synthesize()`) and class static methods (`Synthesizer.getCorpusProjectionAdmission()`, `Synthesizer.buildStaleAssignmentCandidates()`, etc.), so a single class-owned `static #configSnapshot = null` backs both `this.configSnapshot` and static/instance config reads without module-level state.

## The Fix

1. Extend `VectorService.createConfigSnapshot()` in `ai/services/knowledge-base/VectorService.mjs` with all `VectorService`-consumed leaves (`defaultRepoSlug`, `defaultTenantId`, `defaultVisibility`, `embeddingModel`, `embeddingProvider`, `knowledgeBase`, `localModels.embedding`, `logPath`, `mcpSyncMaxChunks`, `ollama`, `openAiCompatible`, `spoofRejectionMode`, `undeliverableTimeoutStrikes`, `vectorDimension`) and route `#resolveLogPath()`, `#getHierarchyPath()`, `resolveTenantStamp()`, `getTenantIsolationConfig()`, `resolveEmbeddingGuardrail()`, `resolveEmbeddingPoisonGeneration()`, `generateIsolationEmbeddings()`, `embed()`, and `embedChunks()` through `#getConfig()`, reading the Tier-1 leaves (`embeddingProvider`, `getEmbeddingModel(provider, cfg)`, `ollama.embeddingTimeoutMs`, `openAiCompatible.batchEmbeddingTimeoutMs`) from KB `aiConfig` rather than importing cross-server `mcConfig`.
2. Add `configSnapshot`, `beforeSetConfigSnapshot()`, and `static createConfigSnapshot()` / `createConfigSnapshot()` to `ai/services/graph/GoldenPathSynthesizer.mjs` backed by `static #configSnapshot = null` and `validateConfigSnapshot` / `cloneConfigSubtree` from `ai/services/shared/configSnapshot.mjs`, routing static and instance `aiConfig` reads through `#getStaticConfig()` / `#getConfig()` and removing duplicate `aiConfig.goldenPath*` parameter defaults from `ai/services/graph/activePrCycleSection.mjs` and `ai/services/graph/issueFocusSections.mjs`.
3. Convert all `7` remaining `VectorService` specs and `GoldenPathSynthesizer.spec.mjs` (`8` test files, `102` writes) to inject `createConfigSnapshot()`, assert `findSharedConfigMutations(source) === []` in each spec, and tighten `ai/scripts/lint/shared-config-write-ceilings.json` (`53 -> 45` files).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `VectorService.createConfigSnapshot()` (`ai/services/knowledge-base/VectorService.mjs`) | ADR-0019 §4 B4 + `#1090` | Clones all `VectorService`-consumed leaves (`defaultRepoSlug`, `defaultTenantId`, `defaultVisibility`, `embeddingModel`, `embeddingProvider`, `knowledgeBase`, `localModels.embedding`, `logPath`, `mcpSyncMaxChunks`, `ollama`, `openAiCompatible`, `spoofRejectionMode`, `undeliverableTimeoutStrikes`, `vectorDimension`) from KB `aiConfig` and routes `#resolveLogPath`, `#getHierarchyPath`, `resolveTenantStamp`, `getTenantIsolationConfig`, `resolveEmbeddingGuardrail`, `resolveEmbeddingPoisonGeneration`, `generateIsolationEmbeddings`, `embed`, and `embedChunks` through `#getConfig()` | Canonical `aiConfig` singleton when `configSnapshot` is `null` | JSDoc on `VectorService` | `npm run test-unit` across all `8` `VectorService*.spec.mjs` files |
| `GoldenPathSynthesizer.configSnapshot` & `GoldenPathSynthesizer.createConfigSnapshot()` (`ai/services/graph/GoldenPathSynthesizer.mjs`, `activePrCycleSection.mjs`, `issueFocusSections.mjs`) | ADR-0019 §4 B4 + `#1090` | Injects and validates an isolated config snapshot (`ollama`, `openAiCompatible`, `orchestrator.corpusProjection`, and `GoldenPathSynthesizer` scalar leaves) via class-owned `static #configSnapshot`, with `GoldenPathSynthesizer.#getStaticConfig()` as the sole resolver for `activePrCycleSection` and `issueFocusSections` | Canonical `aiConfig` singleton when `configSnapshot` is `null` | JSDoc on `GoldenPathSynthesizer` | `npm run test-unit -- test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs` |
| `ai/scripts/lint/shared-config-write-ceilings.json` & `ai/scripts/lint/check-aiconfig-test-mutation.mjs` | `#1096` / PR `#1097` + ADR-0019 §4 B4 | Tightens per-file ceiling sidecar from `53` to `45` files (`8` converted files dropped to ceiling `0`) with code-point key sorting in `tightenCeilings()` | Fails build on any shared-config write in converted specs | JSDoc on `tightenCeilings` | `npm run test-unit -- test/playwright/unit/ai/scripts/lint/checkAiConfigTestMutation.spec.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] Extend `VectorService.createConfigSnapshot()` and `#getConfig()` coverage in `ai/services/knowledge-base/VectorService.mjs` and convert the remaining `7` `VectorService` unit specs (`WorkVolumeBranching`, `undeliverableGeometry`, `persistenceNonConvergence`, `tenantStamping`, `leaseYield`, `failureCarry`, `ollamaIngestSeam`) to `0` shared-config writes, pinned by `findSharedConfigMutations` assertions.
- [ ] Add `configSnapshot`, `beforeSetConfigSnapshot()`, and `createConfigSnapshot()` (backed by `ai/services/shared/configSnapshot.mjs` and class-owned `static #configSnapshot`) to `ai/services/graph/GoldenPathSynthesizer.mjs` and convert `GoldenPathSynthesizer.spec.mjs` (`44 -> 0` shared-config writes), pinned by a `findSharedConfigMutations` assertion and a `beforeSetConfigSnapshot` rejection test.
- [ ] Verify `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` drops global B4 shared-config writes by `102` (`545 -> 443 shared-config write(s) across 125 leaf/leaves (421 assignment, 22 Object.assign)` across `53 -> 45` test files in `shared-config-write-ceilings.json`) with `0` gating violations.

## Out of Scope

- Remaining `443` shared-config writes across `45` non-`VectorService` / non-`GoldenPathSynthesizer` test files (`#1098`).

## Avoided Traps

- **Adding per-leaf `*Fn_` seams or unused snapshot keys:** Rejected per `#541` and `#1090`/`#1092`; only leaves read by `VectorService` and `GoldenPathSynthesizer` are snapshotted.
- **Module-level mutable snapshot state:** `GoldenPathSynthesizer` stores `#configSnapshot` as a private static field on the class rather than a module-scoped `let`.
- **Duplicate `aiConfig` parameter defaults across helper modules:** `activePrCycleSection.mjs` and `issueFocusSections.mjs` receive resolved options from `GoldenPathSynthesizer.#getStaticConfig()` rather than re-reading `aiConfig.goldenPath*` defaults directly.

## Related

- Succeeds: `#541` / PR `#1086`, `#1090` / PR `#1092`, `#1096` / PR `#1097`
- Related: `#508`, `#1098`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "VectorService GoldenPathSynthesizer configSnapshot createConfigSnapshot B4 102 writes 545 443"


## Timeline

- 2026-10-18T00:04:08Z @neo-gemini-pro added the `enhancement` label
- 2026-10-18T00:04:08Z @neo-gemini-pro added the `ai` label
- 2026-10-18T00:04:08Z @neo-gemini-pro added the `refactoring` label
- 2026-10-18T00:04:08Z @neo-gemini-pro added the `testing` label
- 2026-10-18T00:04:09Z @neo-gemini-pro added the `architecture` label
- 2026-10-18T00:06:08Z @neo-gemini-pro cross-referenced by PR #1092
- 2026-10-18T01:41:11Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-18T02:19:54Z @neo-opus-grace cross-referenced by #1096
- 2026-10-18T02:20:04Z @neo-opus-grace added sub-issue #1096
- 2026-10-18T02:32:52Z @neo-opus-grace cross-referenced by PR #1097
- 2026-10-18T02:34:47Z @neo-gemini-pro cross-referenced by #1098
- 2026-10-18T02:35:21Z @neo-gemini-pro changed title from **Convert remaining non-TextEmbeddingService B4 shared-config write concentrations** to **Convert VectorService and GoldenPathSynthesizer B4 shared-config test writes**
- 2026-10-18T02:38:12Z @neo-gemini-pro cross-referenced by PR #1099
- 2026-10-18T02:52:57Z @neo-gemini-pro referenced in commit `8c72b9b` - "fix(ai): align providerLaneComposition consumer census anchors with configSnapshot reads (#1091)"
- 2026-10-18T03:25:04Z @neo-gemini-pro referenced in commit `79c336b` - "fix(ai): address Round-1 review on VectorService and GoldenPathSynthesizer snapshots (#1091)"
- 2026-10-18T03:49:55Z @tobiu referenced in commit `9b01acf` - "feat(ai): convert VectorService and GoldenPathSynthesizer B4 test writes (#1091) (#1099)

* feat(ai): convert VectorService and GoldenPathSynthesizer B4 test writes (#1091)

* fix(ai): align providerLaneComposition consumer census anchors with configSnapshot reads (#1091)

* fix(ai): address Round-1 review on VectorService and GoldenPathSynthesizer snapshots (#1091)"
- 2026-10-18T03:49:55Z @tobiu closed this issue
- 2026-10-18T04:08:30Z @neo-gemini-pro cross-referenced by #1102

