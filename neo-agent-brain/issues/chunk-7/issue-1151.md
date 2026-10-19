---
id: 1151
title: 'Remove remaining configSnapshot usages, delete helpers, and gate in lint'
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
createdAt: '2026-10-18T20:50:41Z'
updatedAt: '2026-10-19T17:40:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1151'
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
  - '[x] 1150 Remove configSnapshot from Memory Core and Knowledge Base services and specs'
blocking: []
closedAt: '2026-10-19T17:40:49Z'
---
# Remove remaining configSnapshot usages, delete helpers, and gate in lint

## Context

Graduated from `neomjs/neo#19856` (`DC_kwDODSospM4BIkDB`, `DC_kwDODSospM4BIkQ2`, `DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2C`, `DC_kwDODSospM4BIk2r`) as **Ticket 3** of the 3-ticket `configSnapshot` removal sequence under Epic `neomjs/neo-agent-brain#1148` (unblocked now that Ticket 1 `#1157` / PR `#1160` and Ticket 2 `#1150` / PR `#1187` are merged).

## The Problem

Following `neomjs/neo-agent-brain#1150` (`#1187`), 12 production service modules in `src/evolution/`, `ai/services/graph/`, and `ai/services/github-workflow/` and 10 unit spec files still carry the parallel `configSnapshot` / `createConfigSnapshot()` / `#propagateConfigSnapshot()` / `this.#configSnapshot || aiConfig` indirection introduced by `#1099` and `#1103`, and the two helper modules (`ai/services/shared/configSnapshot.mjs`, `ai/services/github-workflow/shared/configSnapshot.mjs`) remain on `dev`. Additionally, `check-aiconfig-antipatterns.mjs` does not yet mechanically block `configSnapshot` or whole-root `|| aiConfig` / `|| <name>Config` fallback expressions, and `snapshotAiConfig` (`test/playwright/unit/ai/util.mjs`, added in `#1187`) documents "scalar leaves" in JSDoc without refusing object-valued overrides mechanically (`#1187` Round-2 review `5572974054`).

## The Architectural Reality

- `Design authority: "AiConfig and its child providers are the single source of truth for Agent OS config. READ resolved leaves at the use site. Never re-implement, alias, export, pass-along, mutate, or defend against the SSOT."` (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md §2 & §5.4`).
- `Prescription checked: src/evolution/RemDigestion.mjs, ai/services/graph/{GoldenPathSynthesizer,GapInferenceEngine,SemanticGraphExtractor,TopologyInferenceEngine}.mjs, ai/services/github-workflow/{SyncService,LocalFileService}.mjs, ai/services/github-workflow/sync/{IssueSyncer,PullRequestSyncer,DiscussionSyncer,ReleaseNotesSyncer,MetadataManager}.mjs, ai/scripts/lint/check-aiconfig-antipatterns.mjs, test/playwright/unit/ai/util.mjs — owns the concern`.
- Live `origin/dev` census (`git grep -l "configSnapshot" origin/dev -- ai/ src/ test/`):
  - **2 helper modules to delete (`git rm`):** `ai/services/shared/configSnapshot.mjs`, `ai/services/github-workflow/shared/configSnapshot.mjs`.
  - **12 service modules:** `src/evolution/RemDigestion.mjs`, `ai/services/graph/GoldenPathSynthesizer.mjs`, `ai/services/graph/GapInferenceEngine.mjs`, `ai/services/graph/SemanticGraphExtractor.mjs`, `ai/services/graph/TopologyInferenceEngine.mjs`, `ai/services/github-workflow/SyncService.mjs`, `ai/services/github-workflow/LocalFileService.mjs`, `ai/services/github-workflow/sync/DiscussionSyncer.mjs`, `ai/services/github-workflow/sync/IssueSyncer.mjs`, `ai/services/github-workflow/sync/MetadataManager.mjs`, `ai/services/github-workflow/sync/PullRequestSyncer.mjs`, `ai/services/github-workflow/sync/ReleaseNotesSyncer.mjs`.
  - **10 unit specs to migrate:** `test/playwright/unit/src/evolution/RemDigestion.spec.mjs`, `test/playwright/unit/src/evolution/RemDigestion.executeRemCycle.spec.mjs`, `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/DiscussionSyncer.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/IssueSyncer.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/LocalFileService.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/MetadataManager.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/PullRequestSyncer.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/ReleaseNotesSyncer.spec.mjs`, `test/playwright/unit/ai/services/github-workflow/SyncService.Stage2.spec.mjs`.

## The Fix

1. Remove `configSnapshot`, `#configSnapshot`, `#propagateConfigSnapshot`, `createConfigSnapshot()`, `#getConfig()`, and `configSnapshot.mjs` imports from the 12 service modules so they read resolved leaves directly from `AiConfig` / `aiConfig` at use sites.
2. Delete `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs` via `git rm`.
3. Migrate the 10 unit specs per `ADR-0019 §5.4` `(i)–(iv)` (`snapshotAiConfig` from `test/playwright/unit/ai/util.mjs`), and add a mechanical check in `snapshotAiConfig` (`test/playwright/unit/ai/util.mjs`) that refuses non-scalar (plain/proxy object) override values (`#1187` review `5572974054`).
4. Add a zero-baseline `codeMask` rule (`CONFIG-SNAPSHOT`) in `ai/scripts/lint/check-aiconfig-antipatterns.mjs` (and a RED-first unit spec arm in `test/playwright/unit/ai/scripts/lint/checkAiConfigAntipatterns.spec.mjs`) that flags any executable `configSnapshot` token or whole-root `|| aiConfig` / `|| <name>Config` fallback expression across `ai/**` and `src/**`.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `RemDigestion`, `GoldenPathSynthesizer`, `GapInferenceEngine`, `SemanticGraphExtractor`, `TopologyInferenceEngine`, `SyncService`, `LocalFileService`, `IssueSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, `ReleaseNotesSyncer`, `MetadataManager` | `ADR-0019 §2 & §5.1`, `neomjs/neo#19856` | Read resolved leaves directly from `AiConfig` / `aiConfig` at use sites; `configSnapshot`, `#getConfig`, and `#propagateConfigSnapshot` removed | Fail loud via `ConfigProvider` proxy | JSDoc updated | Unit specs green; `0` executable `configSnapshot` occurrences in `ai/**` and `src/**` |
| `ai/services/shared/configSnapshot.mjs` & `ai/services/github-workflow/shared/configSnapshot.mjs` | `neomjs/neo#19856` OQ4 | Deleted via `git rm` | — | — | Files absent from tree |
| `test/playwright/unit/ai/util.mjs` (`snapshotAiConfig`) | `ADR-0019 §5.4 (iv)`, `#1187` review `5572974054` | Refuses non-scalar (plain/proxy object) override values so deep-merge into an existing object leaf cannot persist across `restore()` | Throws `Error` naming the non-scalar leaf path | JSDoc updated | Unit spec coverage |
| `ai/scripts/lint/check-aiconfig-antipatterns.mjs` | `ADR-0019 §3 (B2/B5)`, `neomjs/neo#19856` (`DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2r`) | Zero-baseline `codeMask` rule fails on executable `configSnapshot` or whole-root `\|\| aiConfig` / `\|\| <name>Config` fallback in `ai/**` and `src/**` | Comments (`deltaDigest.mjs:104`) and leaf-level fallbacks unaffected; `shared-config-write-ceilings.json` untouched | Rule comment in `check-aiconfig-antipatterns.mjs` | `checkAiConfigAntipatterns.spec.mjs` RED-first fixture + green live lint run |

**Decision Record:** `Required: ADR 0019` (carried by Ticket 1 `#1157` / PR `#1160` per `neomjs/neo#19856`).
**Decision Record impact:** `depends-on ADR 0019` (completes the `ADR-0019` `configSnapshot` removal and activates the mechanical anti-pattern guard after Ticket 1 `#1160` and Ticket 2 `#1187`).

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
- `neomjs/neo#19856` OQ1 (`check-aiconfig-antipatterns.mjs` `codeMask` rule across `ai/**` and `src/**`) -> AC-4
- `neomjs/neo#19856` OQ4 (Ticket 3: Evolution / Graph / GitHub-Workflow `configSnapshot` removal + helper deletion + guard activation) -> AC-1, AC-2, AC-3, AC-4, AC-5

## Acceptance Criteria

- [ ] AC-1: `configSnapshot`, `#configSnapshot`, `#propagateConfigSnapshot`, `createConfigSnapshot()`, `#getConfig()`, and `configSnapshot.mjs` imports are removed from the 12 remaining service modules (`RemDigestion`, `GoldenPathSynthesizer`, `GapInferenceEngine`, `SemanticGraphExtractor`, `TopologyInferenceEngine`, `SyncService`, `LocalFileService`, `IssueSyncer`, `PullRequestSyncer`, `DiscussionSyncer`, `ReleaseNotesSyncer`, and `MetadataManager`).
- [ ] AC-2: Both `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs` are deleted via `git rm`.
- [ ] AC-3: The remaining 10 unit specs (`RemDigestion.spec.mjs`, `RemDigestion.executeRemCycle.spec.mjs`, `GoldenPathSynthesizer.spec.mjs`, `DiscussionSyncer.spec.mjs`, `IssueSyncer.spec.mjs`, `LocalFileService.spec.mjs`, `MetadataManager.spec.mjs`, `PullRequestSyncer.spec.mjs`, `ReleaseNotesSyncer.spec.mjs`, `SyncService.Stage2.spec.mjs`) are migrated per `ADR-0019 §5.4` `(i)–(iv)` without calling `ConfigProvider.setEnvOverride` and without raising any ceiling in `ai/scripts/lint/shared-config-write-ceilings.json`, and `snapshotAiConfig` (`test/playwright/unit/ai/util.mjs`) refuses non-scalar (plain/proxy object) override values.
- [ ] AC-4: `ai/scripts/lint/check-aiconfig-antipatterns.mjs` enforces a zero-baseline `codeMask` rule across `ai/**` and `src/**` that flags any executable `configSnapshot` token or whole-root `|| aiConfig` / `|| <name>Config` fallback expression (with a RED-first unit spec arm in `checkAiConfigAntipatterns.spec.mjs`), while leaving `deltaDigest.mjs:104` (JSDoc comment), leaf-level `|| aiConfig.<path>` expressions, and `shared-config-write-ceilings.json` untouched.
- [ ] AC-5: All affected unit specs, `node ai/scripts/lint/check-aiconfig-antipatterns.mjs`, and `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` pass.

## Out of Scope

- Leaf-level fallback census (`<x> || aiConfig.<path>`, 23 lines in 15 files outside the `configSnapshot` files — tracked separately per `DC_kwDODSospM4BIk2r` Point 2).
- Retiring `ai/scripts/lint/shared-config-write-ceilings.json` (`37` unrelated spec files, `342` writes — retires only when the last ceiling reaches zero).

## Avoided Traps

- **Matching leaf-level `|| aiConfig.<path>` in the Ticket 3 rule (`DC_kwDODSospM4BIk2r` Point 2):** A regex that matches `|| *Config` with member access would fail 23 existing leaf-level fallback lines in 15 unrelated files on day one; match the **whole-root** operand (`|| aiConfig` / `|| <name>Config` with no `.` member access) alongside `configSnapshot`.
- **Scanning without `codeMask` (`DC_kwDODSospM4BIkQ2` & `DC_kwDODSospM4BIktj` Point 3):** Raw text grep hits `ai/scripts/diagnostics/deltaDigest.mjs:104` (a JSDoc example); `codeMask` strips comments cleanly.

## Related

- Graduated from `neomjs/neo#19856` (`DC_kwDODSospM4BIkDB`, `DC_kwDODSospM4BIkQ2`, `DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2C`, `DC_kwDODSospM4BIk2r`).
- Unblocked by merged Ticket 1 (`#1157` / PR `#1160`) and Ticket 2 (`#1150` / PR `#1187`).
- Supersedes closed `#1102` / `#1116` and reverses `#1099` & `#1103`.
- Coordinated with `#212` and `#193`.

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T20:48:18Z` (`sort: 'created'`, newest `#1148`); no duplicate found.
A2A claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T20:48:22Z`; no competing claim.
Memory Core rationale sweep: `query_raw_memories("configSnapshot ADR-0019 reactive provider")` confirmed `neomjs/neo#19856` convergence (`56de5757-b2d9-44f6-a570-518c06b92515`).
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-pro` in `neomjs/neo-agent-brain`.
Agent OS structure map: verified via `npm run ai:structure-map -- --files --loc` (`src/evolution/`, `ai/services/graph/`, `ai/services/github-workflow/`, `ai/scripts/lint/`); deletes 2 helper modules, creates 0 new files.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: `query_raw_memories("D#19856 Ticket 3 remove configSnapshot RemDigestion GoldenPathSynthesizer github-workflow delete configSnapshot.mjs")`


## Timeline

- 2026-10-18T20:50:43Z @neo-gemini-julia added the `ai` label
- 2026-10-18T20:50:43Z @neo-gemini-julia added the `refactoring` label
- 2026-10-18T20:50:44Z @neo-gemini-julia added the `testing` label
- 2026-10-18T20:50:44Z @neo-gemini-julia added the `architecture` label
- 2026-10-18T20:50:44Z @neo-gemini-julia added the `agent-os` label
- 2026-10-18T20:50:44Z @neo-gemini-julia added the `tech-debt` label
- 2026-10-18T20:51:14Z @neo-gemini-julia marked this issue as being blocked by #1150
- 2026-10-18T20:54:26Z @neo-gemini-julia cross-referenced by #1150
- 2026-10-18T21:59:30Z @neo-opus-grace cross-referenced by #1157
- 2026-10-18T21:59:50Z @neo-opus-grace marked this issue as being blocked by #1157
- 2026-10-19T13:38:58Z @neo-opus-vega cross-referenced by PR #1187
- 2026-10-19T15:59:19Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T16:17:14Z @neo-gemini-julia cross-referenced by PR #1199
- 2026-10-19T16:56:12Z @neo-gemini-julia cross-referenced by PR #1210
- 2026-10-19T17:08:45Z @neo-gemini-julia referenced in commit `3b0dcad` - "refactor(ai): update golden-path consumer anchors in providerLaneComposition (#1151)"
- 2026-10-19T17:40:49Z @tobiu referenced in commit `bbb3b87` - "fix(ai): remove remaining configSnapshot usages and gate in lint (#1151) (#1210)

* fix(ai): remove remaining configSnapshot usages and gate in lint (#1151)

* refactor(ai): update golden-path consumer anchors in providerLaneComposition (#1151)"
- 2026-10-19T17:40:49Z @tobiu closed this issue

