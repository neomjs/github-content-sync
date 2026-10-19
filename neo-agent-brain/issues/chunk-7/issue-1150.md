---
id: 1150
title: Remove configSnapshot from Memory Core and Knowledge Base services and specs
state: CLOSED
labels:
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
  - tech-debt
assignees:
  - neo-gemini-julia
createdAt: '2026-10-18T20:50:39Z'
updatedAt: '2026-10-19T14:23:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1150'
author: neo-gemini-julia
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1157 Rewrite ADR-0019''s configSnapshot prescription and widen its two guards'
blocking:
  - '[x] 1151 Remove remaining configSnapshot usages, delete helpers, and gate in lint'
closedAt: '2026-10-19T14:23:30Z'
---
# Remove configSnapshot from Memory Core and Knowledge Base services and specs

## Context

Graduated from Ideation Sandbox Discussion `neomjs/neo#19856` (*Brain DDD, Local Host-Edge vs Docker Container-Plane Boundaries, and the ADR-0019 configSnapshot Post-Mortem*).

Following `@tobiu`'s architectural review of `#1115` (`#1135`) and `#1102` (`#1116`), the team audited the `configSnapshot` wave (`#541`/`#1086`, `#1090`/`#1097`, `#1091`/`#1099`, `#1098`/`#1103`, plus closed `#1116` and `#1135`) and converged on **Option C + Option E**:
1. **Ticket 1** (`#1157` / `#1160`, owned by `@neo-opus-grace`) rewrote the 6 misleading `configSnapshot` / *"real injection story"* prescription sites in `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` and `ai/scripts/lint/check-aiconfig-test-mutation.mjs`, codified the `(i)–(iv)` test-isolation ladder (with one named `B4` exception for `(iv)` `snapshotAiConfig`), widened `DB_PATH_LEAVES` in Chroma-scoped form, and widened `ai/scripts/lint/check-aiconfig-antipatterns.mjs` to scan both `ai/` and `src/`.
2. **This ticket (Ticket 2 of 3, `#1150`)** removes the `configSnapshot` second configuration authority (`B2`/`B5`) and `#getConfig()` indirection across the **Memory Core and Knowledge Base services** on `origin/dev` (`ai/services/memory-core/TextEmbeddingService.mjs` from `#541`/`#1090` and `ai/services/knowledge-base/VectorService.mjs` from `#1091`; the other MC/KB services were on unmerged/closed `#1115`), and converts their 13 unit specs per Option E `(i)–(iv)`.

**Design authority:** `ADR-0019 §2, §2.1, §3 (B2/B4/B5), §4, §5.1, §5.4` (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md`), operator ruling on `#1115` / `#1102` (2026-10-18), and graduated Discussion `neomjs/neo#19856`.

## The Problem

In `#541`/`#1086`, `#1090`/`#1097`, and `#1091`/`#1099`, `TextEmbeddingService.mjs` (Memory Core) and `VectorService.mjs` (Knowledge Base) were modified to hold a `#configSnapshot = null` backing field with `get`/`set configSnapshot`, a `createConfigSnapshot()` method, and a private `#getConfig()` helper (`return this.#configSnapshot || aiConfig`):

- **Why this violates `ADR-0019`:**
  1. **Second configuration authority (`B2` / `B5`):** `ADR-0019 §2 & §5.1` mandates reading resolved leaves from the `ConfigProvider` (`AiConfig`, `memoryCoreConfig`, `kbConfig`) at the use site. Wrapping reads in `#getConfig()` creates a competing configuration authority in production services even though zero production callers invoke `createConfigSnapshot()`.
  2. **Non-reactive plain-object copy on the test path (`ADR-0019 §2.1`):** `cloneConfigSubtree` (`ai/services/shared/configSnapshot.mjs:75`) copies `ConfigProvider` state into a non-reactive `{}` object and `isPlainSnapshotObject` rejects `ConfigProvider` instances, bypassing reactive `formulas_`, `observeData` subscriptions, and `#validateAgainstOwner` leaf validators whenever a snapshot is active.

## The Architectural Reality

Verified on `origin/dev` @ `455e8927`:

1. **2 Production Services in `ai/services/memory-core/` and `ai/services/knowledge-base/` on `origin/dev`** (plus `ai/scripts/diagnostics/providerLaneComposition.mjs` coordinate anchor):
   - `ai/services/memory-core/TextEmbeddingService.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`, `#resolveOpenAiCompatibleHost()`)
   - `ai/services/knowledge-base/VectorService.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`)
   - *(Note: the other MC/KB services mentioned during `#19856` ideation lived only on unmerged branch `feat/1115-b4-service-mcp-snapshots` and never landed on `dev`.)*
2. **Corresponding Unit Specs & Test Helper (`13` specs + `util.mjs`):**
   - `test/playwright/unit/ai/deploy/OllamaProviderEnvCoordinates.spec.mjs`
   - `test/playwright/unit/ai/deploy/ProviderReadinessEnvCoordinates.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.WorkVolumeBranching.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.batchFailureIsolation.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.failureCarry.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.leaseYield.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.ollamaIngestSeam.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.persistenceNonConvergence.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.tenantStamping.spec.mjs`
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.undeliverableGeometry.spec.mjs`
   - `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.retry.spec.mjs`
   - `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs`
   - `test/playwright/unit/ai/services/memory-core/util.mjs` + `test/playwright/unit/ai/services/memory-core/util.snapshotAiConfig.spec.mjs`
3. **Note on `ai/services/shared/configSnapshot.mjs`:**
   - Keep `ai/services/shared/configSnapshot.mjs` in tree until **Ticket 3 (`#1151`)** removes the remaining consumers in `src/evolution/RemDigestion.mjs`, `ai/services/graph/`, and `ai/services/github-workflow/` and deletes both helper modules.

## The Fix

1. **In `ai/services/memory-core/TextEmbeddingService.mjs` and `ai/services/knowledge-base/VectorService.mjs` (and `ai/scripts/diagnostics/providerLaneComposition.mjs`):**
   - Delete `#configSnapshot`, `get configSnapshot()`, `set configSnapshot()`, `createConfigSnapshot()`, `#getConfig()`, and the `import { cloneConfigSubtree, validateConfigSnapshot } from '../shared/configSnapshot.mjs'` import.
   - Restore direct use-site reads against `aiConfig` per `ADR-0019 §5.1`.
2. **In `test/playwright/unit/ai/services/memory-core/util.mjs` and the 13 unit specs above:**
   - Extend `snapshotAiConfig` in `util.mjs` to accept an optional `{ 'dot.path': nextValue }` override map for existing leaves (refusing unknown leaves) and track an `activeSnapshots` root/nested baseline per config object so discarded inner override calls are restored by the active root's `restore()` thunk.
   - Classify and replace every `configSnapshot` site across the 13 unit specs per Option E's `(i)–(iv)` ladder:
     - **`(i)` Redundant with `UNIT_TEST_MODE` defaults or `= null` / live-provider resets:** delete.
     - **`(ii)` Worker-scope (harness) env:** already bound before first `ConfigProvider` import (`test/playwright/configTemplateResolver.mjs:140`; **never** call unrestorable `ConfigProvider.setEnvOverride` inside a spec).
     - **`(iii)` Pure-operation scalar/config input:** pass/test the input on the pure helper directly (including `resolveTenantIsolationConfig` for nested-vs-flat tenant config precedence) without mutating `ConfigProvider`.
     - **`(iv)` Synchronous / non-overlapping `ConfigProvider` leaf variation when the service's own `ConfigProvider` read is the unit under test:** use `snapshotAiConfig` (`test/playwright/unit/ai/services/memory-core/util.mjs`) with `try/finally` or `afterEach`/`afterAll` restore.
3. **Coordination note (`STEP_BACK` Point 6):**
   - Notify `#193`'s owner (`@neo-gpt-emmy`) at `[lane-claim]` time since these subtractive in-place edits touch `ai/services/memory-core/` and `ai/services/knowledge-base/`.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `TextEmbeddingService` and `KB_VectorService` config reads (`ai/services/memory-core/TextEmbeddingService.mjs`, `ai/services/knowledge-base/VectorService.mjs`, `ai/scripts/diagnostics/providerLaneComposition.mjs`) | `ADR-0019 §2 & §5.1`, `neomjs/neo#19856` | Read resolved leaves directly from `aiConfig` at use sites (and lift `resolveTenantIsolationConfig` as a pure helper); `configSnapshot` getter/setter, `createConfigSnapshot()`, and `#getConfig()` are removed | Fail loud on invalid leaf access via `ConfigProvider` proxy | Service JSDoc updated | `git grep configSnapshot` across `ai/services/{memory-core,knowledge-base}/` returns `0`; unit specs green |
| 13 Memory Core, Knowledge Base, and deploy coordinate unit specs + `util.mjs` (`snapshotAiConfig`) | `ADR-0019 §4 & §5.4`, `neomjs/neo#19856` Option E | Isolate via `(i)` `UNIT_TEST_MODE` by-construction deletion, `(ii)` worker-scope env, `(iii)` direct pure-operation input (`resolveTenantIsolationConfig`), or `(iv)` sequential `snapshotAiConfig` existing-leaf restore with `activeSnapshots` root/nested baseline tracking | `snapshotAiConfig` restore thunk in `finally`/`afterEach`/`afterAll`; refuses unknown/non-existing leaves | `util.mjs` JSDoc + spec comments where `(iv)` is used | All 13 unit specs (`224` tests in MC/KB + `27` in deploy coordinates) + `check-aiconfig-test-mutation.mjs` + `check-aiconfig-antipatterns.mjs` green |

**Decision Record:** `Required: ADR 0019` (carried by Ticket 1 `#1157` / `#1160` per `neomjs/neo#19856`).
**Decision Record impact:** `depends-on ADR 0019` (implements the `ADR-0019` `configSnapshot` removal for Memory Core and Knowledge Base after Ticket 1 `#1160` amended `ADR-0019 §4/§5.4`).

## Signal Ledger
- `claude`: `[GRADUATION_APPROVED]` (family-of-record via §6.4 aggregation)
  - `[GRADUATION_APPROVED by @neo-opus-grace @ 2026-10-18T20:32:39Z]` (`DC_kwDODSospM4BIk2C`)
  - `[GRADUATION_APPROVED by @neo-opus-vega @ 2026-10-18T20:32:39Z]` (`DC_kwDODSospM4BIk2r`)
- `gemini`: `[AUTHOR_SIGNAL by @neo-gemini-pro @ 2026-10-18T20:32:39Z]`
- `gpt`: divergence read completed and folded (`@neo-gpt-sophie` @ `DC_kwDODSospM4BIkQ2`); see `## Unresolved Liveness`

## Unresolved Dissent
None (Point 4 `GRADUATION_DEFERRED` from `DC_kwDODSospM4BIktj` reconciled in full at `2026-10-18T20:32:39Z` and verified in `DC_kwDODSospM4BIk2C`).

## Unresolved Liveness
- `gpt` (`@neo-gpt-sophie`, `@neo-gpt-emmy`, `@neo-gpt`): `participationStatus: active`; `@neo-gpt-sophie` completed the GPT divergence read (`DC_kwDODSospM4BIkQ2`, folded in full) and was reassigned to the operator-directed 3D Stage goal (`neomjs/neo-stage#183` / `#93`, `MESSAGE:8dfebc8f-5f9e-43ae-8601-fe53e7a86ca9`); `STATUS: peer-owned liveness disposition @ DC_kwDODSospM4BIkQ2` (floor-2 + non-author `APPROVED` satisfied by `gemini` + `claude`).
- `kimi` (`@neo-kimi-phoebe`, `@neo-kimi-iris`): `participationStatus: operator_benched`.

## Discussion Criteria Mapping
- `neomjs/neo#19856` OQ1 (`(i)–(iv)` test-isolation ladder) -> AC-2, AC-3
- `neomjs/neo#19856` OQ4 (Ticket 2: Memory Core + Knowledge Base `configSnapshot` removal across MC/KB services + unit specs) -> AC-1, AC-2, AC-3, AC-4

## Acceptance Criteria

- [ ] AC-1: `configSnapshot`, `#configSnapshot`, `createConfigSnapshot()`, `#getConfig()`, and `configSnapshot.mjs` imports are completely removed from `ai/services/memory-core/TextEmbeddingService.mjs` and `ai/services/knowledge-base/VectorService.mjs` (plus the coordinate anchor in `ai/scripts/diagnostics/providerLaneComposition.mjs`), and all config reads resolve directly through `aiConfig` at the use site.
- [ ] AC-2: All `configSnapshot` usages across the 13 Memory Core, Knowledge Base, and deploy coordinate unit specs are replaced per Option E `(i)–(iv)` (with `snapshotAiConfig` in `util.mjs` remaining strictly an existing-leaves helper that supports override maps and `activeSnapshots` root/nested baseline tracking, and `resolveTenantIsolationConfig` tested at rung `(iii)`), without calling `ConfigProvider.setEnvOverride` in any spec and without raising any ceiling in `ai/scripts/lint/shared-config-write-ceilings.json`.
- [ ] AC-3: `npx playwright test -c test/playwright/playwright.config.unit.mjs` passes on all 13 updated unit specs, and `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` + `node ai/scripts/lint/check-aiconfig-antipatterns.mjs` pass with zero errors.
- [ ] AC-4: At `[lane-claim]` time, `#193`'s owner (`@neo-gpt-emmy`) is notified via A2A that these subtractive edits are landing in `ai/services/memory-core/` and `ai/services/knowledge-base/`.

## Out of Scope

- Rewriting `0019-aiconfig-reactive-provider-ssot.md` and `check-aiconfig-test-mutation.mjs` prescriptions (Ticket 1 `#1157` / `#1160`, shipped by `@neo-opus-grace`).
- Removing `configSnapshot` from `src/evolution/RemDigestion.mjs`, `GoldenPathSynthesizer`, `TopologyInferenceEngine`, `SemanticGraphExtractor`, `GapInferenceEngine`, `SyncService`, `LocalFileService`, `IssueSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, `MetadataManager`, `ReleaseNotesSyncer`, deleting `configSnapshot.mjs`, and activating the `codeMask` `configSnapshot` / whole-root fallback lint rule (`#1151`).
- Moving `ai/services/memory-core/` or `ai/services/knowledge-base/` into `src/` (`#212` / `#193`).

## Avoided Traps

- **Blanket-swapping all `configSnapshot` sites to `snapshotAiConfig` (Option D(a)):** Over half of the spec sites are `= null` resets or redundant with `UNIT_TEST_MODE` defaults (`(i)`) and should be deleted outright rather than converted into shared-singleton writes.
- **Calling `ConfigProvider.setEnvOverride` inside a unit test for `(ii)` (`STEP_BACK` Point 7):** `ConfigProvider.setEnvOverride` (`ai/ConfigProvider.mjs:496`) records into `#runtimeEnvOverrides`, which has no delete or clear API; per-test variation must use `(iii)` or `(iv)`.
- **Retiring `shared-config-write-ceilings.json` (`STEP_BACK` Point 4):** The ceiling sidecar gates `342` writes across `37` unrelated spec files (`0` overlap with the `configSnapshot` specs) and must remain intact.

## Related

- Graduated from `neomjs/neo#19856` (`DC_kwDODSospM4BIkDB`, `DC_kwDODSospM4BIkQ2`, `DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2C`, `DC_kwDODSospM4BIk2r`).
- Unblocked by `#1157` (`#1160`).
- Blocks #1151 (Ticket 3).
- Supersedes closed `#1115` / `#1135` and reverses `#541`/`#1086`, `#1090`/`#1097`, and the KB half of `#1091`/`#1099`.
- Coordinated with `#212` and `#193`.

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T20:48:18Z` (`sort: 'created'`, newest `#1148`); no duplicate found.
A2A claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T20:48:22Z`; no competing claim.
Memory Core rationale sweep: `query_raw_memories("configSnapshot ADR-0019 reactive provider")` confirmed `neomjs/neo#19856` convergence (`56de5757-b2d9-44f6-a570-518c06b92515`).
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-pro` in `neomjs/neo-agent-brain`.
Agent OS structure map: verified via `npm run ai:structure-map -- --files --loc` (`ai/services/memory-core/`, `ai/services/knowledge-base/`); no new `.mjs` files created.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: `query_raw_memories("D#19856 Ticket 2 remove configSnapshot memory-core knowledge-base Option E")`



## Timeline

- 2026-10-18T20:50:41Z @neo-gemini-julia added the `ai` label
- 2026-10-18T20:50:41Z @neo-gemini-julia added the `refactoring` label
- 2026-10-18T20:50:41Z @neo-gemini-julia added the `testing` label
- 2026-10-18T20:50:42Z @neo-gemini-julia added the `architecture` label
- 2026-10-18T20:50:42Z @neo-gemini-julia added the `agent-os` label
- 2026-10-18T20:50:42Z @neo-gemini-julia added the `tech-debt` label
- 2026-10-18T20:51:14Z @neo-gemini-julia marked this issue as blocking #1151
- 2026-10-18T20:54:27Z @neo-gemini-julia cross-referenced by #1151
- 2026-10-18T21:59:30Z @neo-opus-grace cross-referenced by #1157
- 2026-10-18T21:59:45Z @neo-opus-grace marked this issue as being blocked by #1157
- 2026-10-18T23:00:19Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-18T23:15:54Z @neo-fable-clio cross-referenced by #732
- 2026-10-19T13:04:02Z @neo-gemini-julia referenced in commit `f056704` - "fix(ai): remove configSnapshot from Memory Core and Knowledge Base (#1150)"
- 2026-10-19T13:08:42Z @neo-gemini-julia cross-referenced by PR #1187
- 2026-10-19T13:24:51Z @neo-gemini-julia referenced in commit `39dcfe4` - "fix(ai): remove configSnapshot from Memory Core and Knowledge Base (#1150)"
- 2026-10-19T14:07:01Z @neo-gemini-julia referenced in commit `af159f3` - "fix(ai): keep snapshotAiConfig existing-leaves-only and isolate undeliverableGeometry overrides (#1150)"
- 2026-10-19T14:23:30Z @tobiu referenced in commit `cf66245` - "fix(ai): remove configSnapshot from Memory Core and Knowledge Base (#1150) (#1187)

* fix(ai): remove configSnapshot from Memory Core and Knowledge Base (#1150)

* fix(ai): keep snapshotAiConfig existing-leaves-only and isolate undeliverableGeometry overrides (#1150)"
- 2026-10-19T14:23:31Z @tobiu closed this issue
- 2026-10-19T17:21:21Z @neo-opus-vega cross-referenced by PR #1210

