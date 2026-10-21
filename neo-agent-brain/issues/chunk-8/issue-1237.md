---
id: 1237
title: Compact one-shot legacy ADR migration scaffolding and re-stamp neo-stage cut
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
  - tech-debt
assignees:
  - neo-gemini-julia
createdAt: '2026-10-21T12:53:52Z'
updatedAt: '2026-10-21T12:53:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1237'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 1010
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
# Compact one-shot legacy ADR migration scaffolding and re-stamp neo-stage cut

## Context

After `neomjs/neo-agent-brain#1234` (`Resolves #1014`, including stacked `#1233` / `#1232`) merged onto `dev` at commit `b2232454deed8b500841435c91f11aa518194528` (`+5,781 / -40`), `@tobiu` challenged the footprint of the one-shot legacy ADR migration scaffolding (`AGENTS.md §self_evolving_systems` — Accretion Defense):

- **727 LOC** (`ai/services/shared/adrAdmissionCatalog.mjs` `+155`, `ai/services/knowledge-base/source/AdrSource.mjs` `+114`, `ai/services/memory-core/GraphService.mjs` `+3`, `ai/services/ingestion/adrDocumentParser.mjs` `+9`, `test/playwright/unit/ai/services/knowledge-base/source/AdrSource.spec.mjs` `+505`) belong to the permanent cross-process SQLite `adr_admission_catalog` runtime bridge (`#1232`).
- **5,054 LOC** (`ai/scripts/migrations/adrMigrationManifest.mjs` `899 LOC`, `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` `2,614 LOC`, `test/playwright/unit/ai/scripts/migrations/migrateLegacyAdrIdentities.spec.mjs` `1,541 LOC`) belong to the one-shot legacy ADR migration for `41` ADR records (`#1014`). By comparison, sibling `ai/scripts/migrations/normalizeGraphIdentities.mjs` is `158` code LOC (`structure-map`).
- Additionally, `@neo-opus-ada`'s Round-3 review (`#1234#pullrequestreview-5598825407`) noted that `neomjs/neo-stage` `origin/dev` (`ca26ffd9` via `neomjs/neo-stage#264`) relocated `0001-the-engine-contract.md` from `learn/decisions/0001-the-engine-contract.md` (`90cf29759158`) to `learn/agentos/decisions/0001-the-engine-contract.md` (`4a580037288a`), so `MIGRATION_CUT_REPOSITORIES['neomjs/neo-stage']` should be re-stamped before the offline cut (`#1010`).

### Gate 0 Sweep Attestations
- `Live latest-open sweep`: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-21T12:51:45Z` (`#1230` down to `#807`); no equivalent found.
- `A2A in-flight claim sweep`: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-21T12:51:30Z`; no competing claim on `adrMigrationManifest.mjs` or `migrateLegacyAdrIdentities.mjs`.
- `MC sweep`: `query_raw_memories({ query: 'one-shot legacy ADR migration LOC bloat migrateLegacyAdrIdentities adrMigrationManifest', limit: 5 })`, 5 results (own `#1234` audit turn `4a8f7494-a062-4941-addd-97143fbda133`), no prior rejection found.
- `Own-assignment sweep`: 0 open issues assigned in `neomjs/neo-agent-brain`, none overlapping.
- `Structure-map sweep`: ran `npm run ai:structure-map -- --files --loc` in `neomjs/neo-agent-brain`; `ai/scripts/migrations` (`15` files, sibling `normalizeGraphIdentities.mjs` `158` codeLoc) is the owning directory.

## The Problem

1. `ai/scripts/migrations/adrMigrationManifest.mjs` (`899 LOC`) repeats a 22-line verbose object literal 41 times (`RAW_41_RECORDS`) for data that is a deterministic table of 41 compact tuples (`[num, slug, custRepo, rev, hash, nodeType, vecKind, vecId, inDeg, outDeg, sharedOwner, refs, movedFrom]`), plus 150+ lines of prose runbook objects.
2. `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` (`2,614 LOC`) and `test/playwright/unit/ai/scripts/migrations/migrateLegacyAdrIdentities.spec.mjs` (`1,541 LOC`) carry verbose boilerplate adapters, repetitive SQL/Chroma helper ceremony, and verbose per-test fixture construction that inflate a one-shot migration tool to `5,054 LOC`.
3. `MIGRATION_CUT_REPOSITORIES['neomjs/neo-stage']` in `ai/scripts/migrations/adrMigrationManifest.mjs:40-74` still stamps the pre-`neomjs/neo-stage#264` cut (`cutRevision: 'eab1dcca'`, `territoryRoot: 'learn/decisions'`, `sourcePath: 'learn/decisions/0001-the-engine-contract.md'`, `contentHash: '90cf29759158'`) instead of current `neomjs/neo-stage` `origin/dev` (`cutRevision: 'ca26ffd9'`, `territoryRoot: 'learn/agentos/decisions'`, `sourcePath: 'learn/agentos/decisions/0001-the-engine-contract.md'`, `contentHash: '4a580037288a'`).

## The Architectural Reality

- `ai/scripts/migrations/adrMigrationManifest.mjs` exports `MIGRATION_MANIFEST_VERSION`, `MIGRATION_CUT_REPOSITORIES`, `OBSERVED_11_CENSUS_ANOMALY_NUMBERS`, `OBSERVED_LIVE_ANOMALY_NUMBERS`, `MOVED_FM_ADR_NUMBERS`, `REVIEWED_41_ADR_MIGRATION_MANIFEST`, and `ADR_MIGRATION_RUNBOOK`.
- `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` exports `MIGRATION_PHASES`, `computeMigrationContentHash`, `createDefaultSourceProbe`, `createDefaultReaderContext`, `createChromaVectorStoreAdapter`, `createChromaKbStoreAdapter`, `composeDefaultMigrationStores`, `normalizeHistoricalOriginReference`, `buildAdrMigrationRecordFromManifestEntry`, `verifyMigrationPreconditions`, `ensureMigrationTables`, `createAdrMigrationBackup`, `restoreAdrMigrationBackup`, `migrateLegacyAdrIdentities`, `reconcileRebuildAndCustodyControls`, `parseArgs`, `getUsageText`, and `runCli`.
- All exported symbols and `AC1`–`AC8` behaviors from `#1014` can be preserved while compacting the 3 files by **at least 3,200+ net LOC** (`~5,054 LOC` → `~1,400–1,750 LOC` total across manifest + script + spec).

## The Fix

1. Compact `ai/scripts/migrations/adrMigrationManifest.mjs` from `899 LOC` to `~150–200 LOC` by encoding the 41 ADR census entries as compact tuples expanded into the exact same frozen `REVIEWED_41_ADR_MIGRATION_MANIFEST` object shape, and update `MIGRATION_CUT_REPOSITORIES['neomjs/neo-stage']` to `cutRevision: 'ca26ffd9'`, `territoryRoot: 'learn/agentos/decisions'`, `sourcePath: 'learn/agentos/decisions/0001-the-engine-contract.md'`, and `contentHash: '4a580037288a'`.
2. Compact `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` from `2,614 LOC` to `~850–1,050 LOC` by consolidating redundant adapter/helper boilerplate while keeping every exported function, CLI flag, and fail-closed `AC1`–`AC8` invariant intact.
3. Compact `test/playwright/unit/ai/scripts/migrations/migrateLegacyAdrIdentities.spec.mjs` from `1,541 LOC` to `~450–600 LOC` using shared fixture builders while retaining full test coverage for `#1014` `AC1`–`AC8` and the re-stamped `neomjs/neo-stage` cut.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `MIGRATION_CUT_REPOSITORIES` & `REVIEWED_41_ADR_MIGRATION_MANIFEST` in `ai/scripts/migrations/adrMigrationManifest.mjs` | `ADR 0006 §§2.6–2.10` / `#1014` / `neomjs/neo-stage#264` (`ca26ffd9`) | Expands 41 compact tuples into the identical frozen `REVIEWED_41_ADR_MIGRATION_MANIFEST` contract and stamps `neomjs/neo-stage` at `ca26ffd9` / `learn/agentos/decisions/0001-the-engine-contract.md` (`4a580037288a`) | Fail-closed precondition refusal on cut/hash mismatch | JSDoc in `adrMigrationManifest.mjs` | Unit tests in `migrateLegacyAdrIdentities.spec.mjs` |
| `migrateLegacyAdrIdentities` & exported helpers in `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` | `ADR 0006 §§2.6–2.10` / `#1014` / `AGENTS.md §self_evolving_systems` | Preserves all exported functions, CLI flags, Chroma/SQLite adapters, backup/restore, and restartable reconciliation in `<= 1,100 LOC` (down from `2,614 LOC`) | Rollback / precondition abort on store or cut failure | JSDoc in `migrateLegacyAdrIdentities.mjs` | Unit tests in `migrateLegacyAdrIdentities.spec.mjs` |

## Decision Record impact

`aligned-with ADR 0006` (`learn/agentos/decisions/0006-adrs-as-graph-queryable-entities.md` `§§2.6–2.10`) and `aligned-with ADR 0019` (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md`).

## Acceptance Criteria

- [ ] **AC-1:** `ai/scripts/migrations/adrMigrationManifest.mjs` is compacted to `<= 220 LOC` (from `899 LOC`) while preserving the exact exported constants and per-entry schema of `REVIEWED_41_ADR_MIGRATION_MANIFEST`, and re-stamps `MIGRATION_CUT_REPOSITORIES['neomjs/neo-stage']` to `cutRevision: 'ca26ffd9'`, `territoryRoot: 'learn/agentos/decisions'`, `sourcePath: 'learn/agentos/decisions/0001-the-engine-contract.md'`, and `contentHash: '4a580037288a'`.
- [ ] **AC-2:** `ai/scripts/migrations/migrateLegacyAdrIdentities.mjs` is compacted to `<= 1,100 LOC` (from `2,614 LOC`) while preserving all exported functions (`MIGRATION_PHASES`, `computeMigrationContentHash`, `createDefaultSourceProbe`, `createDefaultReaderContext`, `createChromaVectorStoreAdapter`, `createChromaKbStoreAdapter`, `composeDefaultMigrationStores`, `normalizeHistoricalOriginReference`, `buildAdrMigrationRecordFromManifestEntry`, `verifyMigrationPreconditions`, `ensureMigrationTables`, `createAdrMigrationBackup`, `restoreAdrMigrationBackup`, `migrateLegacyAdrIdentities`, `reconcileRebuildAndCustodyControls`, `parseArgs`, `getUsageText`, `runCli`) and `#1014` `AC1`–`AC8` semantics.
- [ ] **AC-3:** `test/playwright/unit/ai/scripts/migrations/migrateLegacyAdrIdentities.spec.mjs` is compacted to `<= 650 LOC` (from `1,541 LOC`) while keeping all unit tests passing against both multi-repo checkouts and single-repo CI runners, resulting in a net deletion of `>= 3,200 LOC` across the three files.

## Out of Scope

- Modifying the permanent `adr_admission_catalog` runtime bridge (`ai/services/shared/adrAdmissionCatalog.mjs`, `AdrSource.mjs`, `GraphService.mjs`).
- Executing the live offline production cut against `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite` (tracked on `#1014` `AC8` / parent `#1010`).

## Avoided Traps

- **Deleting the migration script before the live Docker volume cut runs:** The live SQLite graph inside the running Docker stack still holds legacy `adr-0006` until the maintenance-window cut runs, so compacting the implementation now (and retiring the script after `#1010` closes) avoids breaking the pending cut while immediately eliminating `>3,200 LOC` of bloat.

## Related

- Part of `#1010`
- Follow-up to `#1014` / PR `#1234`

Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85
Handoff Retrieval Hints: `query_raw_memories({ query: "compact adrMigrationManifest migrateLegacyAdrIdentities LOC b2232454" })`


## Timeline

- 2026-10-21T12:53:54Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-21T12:53:54Z @neo-gemini-julia added the `enhancement` label
- 2026-10-21T12:53:54Z @neo-gemini-julia added the `ai` label
- 2026-10-21T12:53:55Z @neo-gemini-julia added the `refactoring` label
- 2026-10-21T12:53:55Z @neo-gemini-julia added the `agent-os` label
- 2026-10-21T12:53:55Z @neo-gemini-julia added the `tech-debt` label
- 2026-10-21T12:53:57Z @neo-gemini-julia added parent issue #1010
- 2026-10-21T13:25:24Z @neo-gemini-julia cross-referenced by PR #1238

