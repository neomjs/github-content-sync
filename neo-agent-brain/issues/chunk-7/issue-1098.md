---
id: 1098
title: Convert RemDigestion and github-workflow B4 shared-config test writes
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - testing
  - architecture
assignees:
  - neo-gemini-pro
createdAt: '2026-10-18T02:34:45Z'
updatedAt: '2026-10-18T04:09:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1098'
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
# Convert RemDigestion and github-workflow B4 shared-config test writes

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T03:02:15Z` (`#1095`, `#1091`, `#1081`, `#1048`, `#1047`, `#1014`, `#1013`, `#1012`, `#1010`, `#1005`, `#979`, `#978`, `#977`, `#969`, `#957`, `#936`, `#933`, `#904`, `#875`, `#850`); no duplicate found.
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T03:02:10Z`; no competing claim on converting the `RemDigestion` and `github-workflow` B4 shared-config write concentrations.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"check-aiconfig-test-mutation B4 shared-config write concentrations RemDigestion github-workflow configSnapshot"` at `2026-10-18T03:02:05Z`; surfaced `#541`, `#1086`, `#1090`, `#1092`, `#1091`, and `#1096`.
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` at `2026-10-18T03:02:00Z`; `#1091` was open and has since merged via PR `#1099`.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`src/evolution/RemDigestion.mjs`, `ai/services/graph/*`, `ai/services/github-workflow/*`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## Context

`#541` (PR `#1086`), `#1090` (PR `#1092`), and `#1091` (PR `#1099`) converted `TextEmbeddingService` (`2` specs), `VectorService` (`7` specs), and `GoldenPathSynthesizer` (`1` spec) to isolated `configSnapshot` / `createConfigSnapshot()` injection backed by `ai/services/shared/configSnapshot.mjs`, while `#1096` (PR `#1097`) added the per-file ceiling ratchet (`ai/scripts/lint/shared-config-write-ceilings.json`), reducing global B4 shared-config writes from `655` to `443` across `45` test files.

## The Problem

Within the `443` remaining B4 shared-config writes across `45` test files, `RemDigestion` (`2` specs, `52` writes) and `ai/services/github-workflow/` (`6` specs, `49` writes) account for `101` shared-config writes (`22.8%` of the remaining debt):
- `test/playwright/unit/src/evolution/RemDigestion.spec.mjs` (`44` writes) and `test/playwright/unit/src/evolution/RemDigestion.executeRemCycle.spec.mjs` (`8` writes) directly mutate `AiConfig` and `EvolutionConfig` leaves consumed by `RemDigestion` and its three graph collaborators (`GapInferenceEngine`, `SemanticGraphExtractor`, `TopologyInferenceEngine`).
- `test/playwright/unit/ai/services/github-workflow/DiscussionSyncer.spec.mjs` (`16` writes), `IssueSyncer.spec.mjs` (`11` writes), `ReleaseNotesSyncer.spec.mjs` (`7` writes), `MetadataManager.spec.mjs` (`6` writes), `PullRequestSyncer.spec.mjs` (`6` writes), and `LocalFileService.spec.mjs` (`3` writes) directly mutate `aiConfig.issueSync` and `aiConfig.pullRequest` leaves (`contentRootOverride`, `repo`, `*DirOverride`, `commentSectionHeading`, `badPracticeSectionHeading`, `reviewerMap`).

## The Architectural Reality

- `ai/services/shared/configSnapshot.mjs` (`cloneConfigSubtree`, `validateConfigSnapshot`) and `ai/services/shared/serviceProxy.mjs` (`makeSafe` `set` trap forwarding) already provide the canonical ADR-0019 §4 B4 test-isolation seam.
- `RemDigestion` (`src/evolution/RemDigestion.mjs`) delegates graph analysis to `GapInferenceEngine` (`ai/services/graph/GapInferenceEngine.mjs`), `SemanticGraphExtractor` (`ai/services/graph/SemanticGraphExtractor.mjs`), and `TopologyInferenceEngine` (`ai/services/graph/TopologyInferenceEngine.mjs`); setting `RemDigestion.configSnapshot` must cascade the `aiConfig` snapshot to all three collaborators so tests configuring `RemDigestion` isolate the entire REM cycle.
- `aiConfig.issueSync` (`ai/mcp/server/github-workflow/configBase.mjs`) defines `10` reactive path getters (`metadataBaseRoot`, `contentRoot`, `corpusLeaseFile`, `originRoot`, `issuesDir`, `archiveRoot`, `discussionsDir`, `pullsDir`, `metadataFile`, `releaseNotesDir`) derived from `contentRootOverride`, `repo`, and `*DirOverride`. A shared `ai/services/github-workflow/shared/configSnapshot.mjs` helper (`createGitHubWorkflowConfigSnapshot`, `validateGitHubWorkflowConfigSnapshot`) can preserve those live getter formulas on cloned snapshots so overriding `contentRootOverride` or `repo` in `createConfigSnapshot()` recomputes all derived paths deterministically.

## The Fix

1. Add `configSnapshot: null`, `beforeSetConfigSnapshot(value)` (via `validateConfigSnapshot`), `createConfigSnapshot(overrides = {})`, and `#getConfig()` to `RemDigestion` (`src/evolution/RemDigestion.mjs`, plus `#getEvolutionConfig()` and uniform collaborator cascade to `GapInferenceEngine`, `SemanticGraphExtractor`, and `TopologyInferenceEngine`) and to `GapInferenceEngine`, `SemanticGraphExtractor`, and `TopologyInferenceEngine`.
2. Create `ai/services/github-workflow/shared/configSnapshot.mjs` (`createGitHubWorkflowConfigSnapshot`, `validateGitHubWorkflowConfigSnapshot`) and wire `configSnapshot`, `beforeSetConfigSnapshot`, `createConfigSnapshot(overrides = {})`, and `#getConfig()` into `LocalFileService`, `MetadataManager`, `ReleaseNotesSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, and `IssueSyncer`.
3. Convert all `8` specs (`RemDigestion.spec.mjs`, `RemDigestion.executeRemCycle.spec.mjs`, `LocalFileService.spec.mjs`, `MetadataManager.spec.mjs`, `ReleaseNotesSyncer.spec.mjs`, `PullRequestSyncer.spec.mjs`, `DiscussionSyncer.spec.mjs`, `IssueSyncer.spec.mjs` — `101` writes total) to `createConfigSnapshot()` injection, add a `findSharedConfigMutations(source) === []` unit test to each spec, and run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` to remove all `8` files from `ai/scripts/lint/shared-config-write-ceilings.json` (`45 -> 37` files, `443 -> 342` writes).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `RemDigestion.configSnapshot` / `RemDigestion.createConfigSnapshot(overrides)` (`src/evolution/RemDigestion.mjs`) | ADR-0019 §4 B4 (`0019-aiconfig-reactive-provider-ssot.md`) | Isolates `aiConfig` + `evolutionConfig` leaves per test and cascades the `aiConfig` snapshot to `GapInferenceEngine`, `SemanticGraphExtractor`, and `TopologyInferenceEngine` | `null` reads module-imported `AiConfig` and `EvolutionConfig` | JSDoc on `RemDigestion` | `RemDigestion.spec.mjs` + `RemDigestion.executeRemCycle.spec.mjs` |
| `GapInferenceEngine` / `SemanticGraphExtractor` / `TopologyInferenceEngine` `configSnapshot` + `createConfigSnapshot(overrides)` (`ai/services/graph/*.mjs`) | ADR-0019 §4 B4 | Isolates `aiConfig` leaves (`data`, `remSleep`, `graphProvider`, `ollama`, `openAiCompatible`, `geminiApiKey`) per test | `null` reads module-imported `AiConfig` | JSDoc on each class | `RemDigestion.spec.mjs` + `RemDigestion.executeRemCycle.spec.mjs` |
| `createGitHubWorkflowConfigSnapshot` / `validateGitHubWorkflowConfigSnapshot` (`ai/services/github-workflow/shared/configSnapshot.mjs`) + `configSnapshot` on `LocalFileService`, `MetadataManager`, `ReleaseNotesSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, `IssueSyncer` | ADR-0019 §4 B4 | Clones `issueSync` and `pullRequest` subtrees while preserving reactive path getters (`contentRoot`, `issuesDir`, `discussionsDir`, `pullsDir`, `archiveRoot`, `metadataFile`, `releaseNotesDir`, `originRoot`, `metadataBaseRoot`, `corpusLeaseFile`) | `null` reads module-imported `aiConfig` | JSDoc on helper + 6 services | `6` `github-workflow` unit specs |
| `ai/scripts/lint/shared-config-write-ceilings.json` | `#1096` / PR `#1097` | Ratchets from `45` files (`443` writes) to `37` files (`342` writes across `104` leaves: `320` assignment, `22` `Object.assign`) | Per-file ceiling enforcement (`0` for unlisted specs) | `check-aiconfig-test-mutation.mjs` | `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] Add `configSnapshot`, `beforeSetConfigSnapshot`, `createConfigSnapshot(overrides = {})`, and `#getConfig()` to `RemDigestion`, `GapInferenceEngine`, `SemanticGraphExtractor`, `TopologyInferenceEngine`, `LocalFileService`, `MetadataManager`, `ReleaseNotesSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, and `IssueSyncer` (backed by `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs`), with no per-leaf `*Fn_` seams.
- [ ] Convert all `8` target specs (`101` writes) to `createConfigSnapshot()` injection, assert `findSharedConfigMutations(source) === []` in each spec, and tighten `ai/scripts/lint/shared-config-write-ceilings.json` from `45 -> 37` files (`443 -> 342` writes).

## Out of Scope

- Remaining `342` B4 shared-config writes across `37` daemon/service test files (`Orchestrator`, `DeploymentStateBridgeService`, `TenantRepoSyncService`, `SessionService`, `DestructiveOperationGuard`, KB sources, etc.), tracked in `#1102`.

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`, `#1090`, and `#1091`; tests inject an isolated `configSnapshot` rather than proliferating per-leaf function hooks.
- **Static value-copying of `aiConfig.issueSync` getters:** A plain `structuredClone` freezes `contentRoot` / `issuesDir` at clone time before `contentRootOverride` overrides are applied; `createGitHubWorkflowConfigSnapshot` re-attaches the reactive path getters so overrides recompute derived paths accurately.

## Related

- Succeeds: `#541` / PR `#1086`, `#1090` / PR `#1092`, `#1091` / PR `#1099`
- Succeeded by: `#1102`
- Related: `#508`, `#1096` / PR `#1097`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "findSharedConfigMutations B4 RemDigestion github-workflow createGitHubWorkflowConfigSnapshot 342 shared-config writes"


## Timeline

- 2026-10-18T02:34:46Z @neo-gemini-pro added the `enhancement` label
- 2026-10-18T02:34:47Z @neo-gemini-pro added the `ai` label
- 2026-10-18T02:34:47Z @neo-gemini-pro added the `refactoring` label
- 2026-10-18T02:34:47Z @neo-gemini-pro added the `testing` label
- 2026-10-18T02:34:47Z @neo-gemini-pro added the `architecture` label
- 2026-10-18T02:38:12Z @neo-gemini-pro cross-referenced by PR #1099
- 2026-10-18T02:40:54Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-18T04:08:30Z @neo-gemini-pro cross-referenced by #1102
- 2026-10-18T04:09:31Z @neo-gemini-pro changed title from **Convert remaining Orchestrator, RemDigestion, and daemon B4 shared-config writes** to **Convert RemDigestion and github-workflow B4 shared-config test writes**
- 2026-10-18T04:11:52Z @neo-gemini-pro cross-referenced by PR #1103
- 2026-10-18T04:23:10Z @neo-gemini-pro referenced in commit `e9dfa73` - "feat(ai): preserve pullFilenamePrefix config access in PullRequestSyncer (#1098)"

