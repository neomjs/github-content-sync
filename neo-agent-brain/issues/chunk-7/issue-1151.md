---
id: 1151
title: 'Remove remaining configSnapshot usages, delete helpers, and gate in lint'
state: OPEN
labels:
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
  - tech-debt
assignees: []
createdAt: '2026-10-18T20:50:41Z'
updatedAt: '2026-10-18T20:54:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1151'
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
blockedBy:
  - '[x] 1157 Rewrite ADR-0019''s configSnapshot prescription and widen its two guards'
  - '[ ] 1150 Remove configSnapshot from Memory Core and Knowledge Base services and specs'
blocking: []
---
# Remove remaining configSnapshot usages, delete helpers, and gate in lint

## Context

Graduated from Ideation Sandbox Discussion `neomjs/neo#19856` (*Brain DDD, Local Host-Edge vs Docker Container-Plane Boundaries, and the ADR-0019 configSnapshot Post-Mortem*).

This is **Ticket 3 of 3 (`#1151`)** in the converged **Option C + Option E** sequence from `neomjs/neo#19856`:
1. **Ticket 1** (owned by `@neo-opus-grace`) rewrites the 6 misleading `configSnapshot` / *"real injection story"* prescription sites in `0019-aiconfig-reactive-provider-ssot.md` and `check-aiconfig-test-mutation.mjs`, codifies the `(i)–(iv)` test-isolation ladder, widens `DB_PATH_LEAVES` in Chroma-scoped form, and widens `check-aiconfig-antipatterns.mjs` (`:346` and `:359`) to scan both `ai/` and `src/`.
2. **Ticket 2 (`#1150`)** removes `configSnapshot` from the 8 Memory Core and Knowledge Base services and their 12 unit specs (`#1086`, `#1092`).
3. **This ticket (Ticket 3 of 3, `#1151`)** removes `configSnapshot` from the remaining **6 Evolution, Graph, and GitHub Workflow classes** and their **9 unit specs** (`#1099`, `#1103`), deletes both `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs`, and activates the zero-tolerance `codeMask` rule in `ai/scripts/lint/check-aiconfig-antipatterns.mjs` forbidding `configSnapshot` and whole-root `|| aiConfig` / `|| <name>Config` fallback wrappers across `ai/**` and `src/**`.

**Design authority:** `ADR-0019 §2, §2.1, §3 (B2/B4/B5), §4, §5.1, §5.4` (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md`), operator ruling on `#1115` / `#1102` (2026-10-18), and graduated Discussion `neomjs/neo#19856`.

## The Problem

In `#1099` and `#1103`, `configSnapshot` was extended into `src/evolution/RemDigestion.mjs`, `GoldenPathSynthesizer`, `StorageRouter`, `IssueSyncer`, `PullRequestSyncer`, and `DiscussionSyncer`:

1. **`B5` propagation inside `src/`:** `src/evolution/RemDigestion.mjs:326,331-332` actively forwards `collaborator.configSnapshot = next` into its collaborators (`GoldenPathSynthesizer`, etc.), infecting the domain-extracted `src/evolution/` package (`#215`) with a second configuration authority.
2. **Shared snapshot cloner & builder modules:** `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs` (plus their dedicated unit specs `configSnapshot.spec.mjs` and `githubWorkflowConfigSnapshot.spec.mjs`) exist solely to construct flat, non-reactive plain-object copies that reject `ConfigProvider` instances (`isPlainSnapshotObject`).
3. **Missing mechanical reintroduction guard:** Without a zero-baseline rule in `ai/scripts/lint/check-aiconfig-antipatterns.mjs`, a future PR could reintroduce `configSnapshot` or whole-root `|| aiConfig` / `|| <name>Config` fallback wrappers in `ai/**` or `src/**`.

## The Architectural Reality

Verified on `origin/dev` @ `9cd1528b`:

1. **6 Production Classes + 2 Shared Snapshot Modules:**
   - `src/evolution/RemDigestion.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `#propagateConfigSnapshot()` at `:326,331-332`, `createConfigSnapshot()`, `#getConfig()`)
   - `ai/services/memory-core/GoldenPathSynthesizer.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`)
   - `ai/services/graph/StorageRouter.mjs` (`configSnapshot = null`, `#getConfig()`)
   - `ai/services/github-workflow/IssueSyncer.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`)
   - `ai/services/github-workflow/PullRequestSyncer.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`)
   - `ai/services/github-workflow/DiscussionSyncer.mjs` (`#configSnapshot = null`, `get`/`set configSnapshot`, `createConfigSnapshot()`, `#getConfig()`)
   - `ai/services/shared/configSnapshot.mjs` (**delete via `git rm`**)
   - `ai/services/github-workflow/shared/configSnapshot.mjs` (**delete via `git rm`**)
2. **9 Unit Specs (`test/playwright/unit/ai/`):**
   - `test/playwright/unit/ai/services/shared/configSnapshot.spec.mjs` (**delete via `git rm`** — tests only the deleted helper)
   - `test/playwright/unit/ai/services/github-workflow/githubWorkflowConfigSnapshot.spec.mjs` (**delete via `git rm`** — tests only the deleted GitHub snapshot builder)
   - `test/playwright/unit/ai/services/memory-core/RemDigestion.Atomicity.spec.mjs`
   - `test/playwright/unit/ai/services/memory-core/RemDigestion.Judge.spec.mjs`
   - `test/playwright/unit/ai/services/memory-core/GoldenPathSynthesizer.spec.mjs`
   - `test/playwright/unit/ai/services/graph/StorageRouter.spec.mjs`
   - `test/playwright/unit/ai/services/github-workflow/IssueSyncer.SyncState.spec.mjs`
   - `test/playwright/unit/ai/services/github-workflow/PullRequestSyncer.SyncState.spec.mjs`
   - `test/playwright/unit/ai/services/github-workflow/DiscussionSyncer.SubtleDefect.spec.mjs`
3. **Whole-Root vs. Leaf-Level Fallback Distinction (`@neo-opus-vega` `DC_kwDODSospM4BIk2r` Point 2 + `@neo-gpt-sophie` `DC_kwDODSospM4BIkQ2`):**
   - `ai/scripts/diagnostics/deltaDigest.mjs:104` has a JSDoc-only comment match for `configSnapshot`; `codeMask` (`ai/scripts/lint/check-aiconfig-antipatterns.mjs:13`) strips comments automatically so `deltaDigest.mjs:104` is unaffected.
   - Across `ai/**` and `src/**` (excluding comments), **whole-root fallback** (`|| aiConfig` or `|| <name>Config` with no member access) has **14 lines on `dev`, all on `configSnapshot` lines, and `0` elsewhere**, whereas leaf-level fallback (`<x> || aiConfig.<path>`) has `23` lines in `15` files outside the 39 (e.g. Fleet services' `this.dataDir || aiConfig.fleet.dataDir`, `Orchestrator.mjs:1666`, migrations' `options.db || aiConfig.storagePaths.graph`). The new `check-aiconfig-antipatterns.mjs` rule MUST therefore target `configSnapshot` tokens and **whole-root** `|| aiConfig` / `|| <name>Config` fallback expressions in `codeMask`-filtered code.
   - As verified by `@neo-opus-grace` (`DC_kwDODSospM4BIktj` Point 4), `ai/scripts/lint/shared-config-write-ceilings.json` (`37` files, `342` writes, `0` overlap with the `21` `configSnapshot` specs) MUST remain intact until the last ceiling reaches zero.

## The Fix

1. **Remove `configSnapshot` from the 6 classes above** (`RemDigestion`, `GoldenPathSynthesizer`, `StorageRouter`, `IssueSyncer`, `PullRequestSyncer`, `DiscussionSyncer`), including `RemDigestion.#propagateConfigSnapshot()`, and restore direct use-site reads against `aiConfig`, `memoryCoreConfig`, or `githubWorkflowConfig`.
2. **Update the 7 service unit specs per Option E `(i)–(iv)`** and **`git rm` the 2 snapshot helper modules and their 2 dedicated unit specs** (`ai/services/shared/configSnapshot.mjs`, `ai/services/github-workflow/shared/configSnapshot.mjs`, `configSnapshot.spec.mjs`, `githubWorkflowConfigSnapshot.spec.mjs`).
3. **Add the zero-baseline `codeMask` rule in `ai/scripts/lint/check-aiconfig-antipatterns.mjs`:**
   - Scan `codeMask(source)` across `ai/**` and `src/**` (relying on the `:346` and `:359` `src/` scan-root widening from Ticket 1) and fail on any executable `configSnapshot` identifier or whole-root `|| aiConfig` / `|| <name>Config` fallback expression (no member access on the right-hand operand).
   - Add a unit test arm in `test/playwright/unit/ai/scripts/lint/checkAiConfigAntipatterns.spec.mjs` proving RED on a `src/` and `ai/` `configSnapshot` / `|| aiConfig` fixture while ignoring JSDoc comments (`deltaDigest.mjs:104`) and leaf-level `options.x || aiConfig.y.z` expressions.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `RemDigestion`, `GoldenPathSynthesizer`, `StorageRouter`, `IssueSyncer`, `PullRequestSyncer`, `DiscussionSyncer` | `ADR-0019 §2 & §5.1`, `neomjs/neo#19856` | Read resolved leaves directly from `aiConfig` / `memoryCoreConfig` / `githubWorkflowConfig` at use sites; `configSnapshot` and `#propagateConfigSnapshot` removed | Fail loud via `ConfigProvider` proxy | JSDoc updated | Unit specs green; `0` executable `configSnapshot` occurrences in `ai/**` and `src/**` |
| `ai/services/shared/configSnapshot.mjs` & `ai/services/github-workflow/shared/configSnapshot.mjs` | `neomjs/neo#19856` OQ4 | Deleted via `git rm` along with `configSnapshot.spec.mjs` and `githubWorkflowConfigSnapshot.spec.mjs` | — | — | Files absent from tree |
| `ai/scripts/lint/check-aiconfig-antipatterns.mjs` | `ADR-0019 §3 (B2/B5)`, `neomjs/neo#19856` (`DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2r`) | Zero-baseline `codeMask` rule fails on executable `configSnapshot` or whole-root `\|\| aiConfig` / `\|\| <name>Config` fallback in `ai/**` and `src/**` | Comments (`deltaDigest.mjs:104`) and leaf-level fallbacks unaffected; `shared-config-write-ceilings.json` untouched | Rule comment in `check-aiconfig-antipatterns.mjs` | `checkAiConfigAntipatterns.spec.mjs` RED-first fixture + green live lint run |

**Decision Record:** `Required: ADR 0019` (carried by Ticket 1 per `neomjs/neo#19856`; if Ticket 1's `(i)–(iv)` ladder + boundary text does not net-reduce bytes, it cites decay-mitigation rationale: strike the named `(iv)` exception when its last use goes).
**Decision Record impact:** `depends-on ADR 0019` (completes the `ADR-0019` `configSnapshot` removal and activates the mechanical anti-pattern guard after Ticket 1 and Ticket 2).

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

- [ ] AC-1: `configSnapshot`, `#configSnapshot`, `#propagateConfigSnapshot`, `createConfigSnapshot()`, `#getConfig()`, and `configSnapshot.mjs` imports are removed from `src/evolution/RemDigestion.mjs`, `GoldenPathSynthesizer`, `StorageRouter`, `IssueSyncer`, `PullRequestSyncer`, and `DiscussionSyncer`.
- [ ] AC-2: Both `ai/services/shared/configSnapshot.mjs` and `ai/services/github-workflow/shared/configSnapshot.mjs` and their dedicated specs (`configSnapshot.spec.mjs`, `githubWorkflowConfigSnapshot.spec.mjs`) are deleted via `git rm`.
- [ ] AC-3: The remaining 7 unit specs (`RemDigestion.Atomicity`, `RemDigestion.Judge`, `GoldenPathSynthesizer`, `StorageRouter`, `IssueSyncer.SyncState`, `PullRequestSyncer.SyncState`, `DiscussionSyncer.SubtleDefect`) are migrated per Option E `(i)–(iv)` without calling `ConfigProvider.setEnvOverride` and without raising any ceiling in `ai/scripts/lint/shared-config-write-ceilings.json`.
- [ ] AC-4: `ai/scripts/lint/check-aiconfig-antipatterns.mjs` enforces a zero-baseline `codeMask` rule across `ai/**` and `src/**` that flags any executable `configSnapshot` token or whole-root `|| aiConfig` / `|| <name>Config` fallback expression (with a RED-first unit spec arm in `checkAiConfigAntipatterns.spec.mjs`), while leaving `deltaDigest.mjs:104` (JSDoc comment), leaf-level `|| aiConfig.<path>` expressions, and `shared-config-write-ceilings.json` untouched.
- [ ] AC-5: All affected unit specs, `node ai/scripts/lint/check-aiconfig-antipatterns.mjs`, and `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` pass.

## Out of Scope

- Leaf-level fallback census (`<x> || aiConfig.<path>`, 23 lines in 15 files outside the 39 `configSnapshot` files — tracked separately per `DC_kwDODSospM4BIk2r` Point 2).
- Retiring `ai/scripts/lint/shared-config-write-ceilings.json` (`37` unrelated spec files, `342` writes — retires only when the last ceiling reaches zero).

## Avoided Traps

- **Matching leaf-level `|| aiConfig.<path>` in the Ticket 3 rule (`DC_kwDODSospM4BIk2r` Point 2):** A regex that matches `|| *Config` with member access would fail 23 existing leaf-level fallback lines in 15 unrelated files on day one; match the **whole-root** operand (`|| aiConfig` / `|| <name>Config` with no `.` member access) alongside `configSnapshot`.
- **Scanning without `codeMask` (`DC_kwDODSospM4BIkQ2` & `DC_kwDODSospM4BIktj` Point 3):** Raw text grep hits `ai/scripts/diagnostics/deltaDigest.mjs:104` (a JSDoc example); `codeMask` strips comments cleanly.

## Related

- Graduated from `neomjs/neo#19856` (`DC_kwDODSospM4BIkDB`, `DC_kwDODSospM4BIkQ2`, `DC_kwDODSospM4BIktj`, `DC_kwDODSospM4BIk2C`, `DC_kwDODSospM4BIk2r`).
- Blocked by Ticket 1 (`@neo-opus-grace`) and Ticket 2 (#1150).
- Supersedes closed `#1102` / `#1116` and reverses `#1099` & `#1103`.
- Coordinated with `#212` and `#193`.

unowned-rationale: graduated from `neomjs/neo#19856` as Ticket 3 of the 3-ticket `configSnapshot` removal sequence (sequenced after Ticket 1 and Ticket 2 `#1150`); open for pickup by `@neo-gemini-pro`, `@neo-opus-grace`, `@neo-opus-ada`, or `@neo-fable-clio` once Tickets 1 and 2 land.

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T20:48:18Z` (`sort: 'created'`, newest `#1148`); no duplicate found.
A2A claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T20:48:22Z`; no competing claim.
Memory Core rationale sweep: `query_raw_memories("configSnapshot ADR-0019 reactive provider")` confirmed `neomjs/neo#19856` convergence (`56de5757-b2d9-44f6-a570-518c06b92515`).
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-pro` in `neomjs/neo-agent-brain`.
Agent OS structure map: verified via `npm run ai:structure-map -- --files --loc` (`src/evolution/`, `ai/services/graph/`, `ai/services/github-workflow/`, `ai/scripts/lint/`); deletes 2 helper modules and 2 specs, creates 0 new files.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: `query_raw_memories("D#19856 Ticket 3 remove configSnapshot RemDigestion StorageRouter github-workflow delete configSnapshot.mjs")`


## Timeline

- 2026-10-18T20:50:43Z @neo-gemini-pro added the `ai` label
- 2026-10-18T20:50:43Z @neo-gemini-pro added the `refactoring` label
- 2026-10-18T20:50:44Z @neo-gemini-pro added the `testing` label
- 2026-10-18T20:50:44Z @neo-gemini-pro added the `architecture` label
- 2026-10-18T20:50:44Z @neo-gemini-pro added the `agent-os` label
- 2026-10-18T20:50:44Z @neo-gemini-pro added the `tech-debt` label
- 2026-10-18T20:51:14Z @neo-gemini-pro marked this issue as being blocked by #1150
- 2026-10-18T20:54:26Z @neo-gemini-pro cross-referenced by #1150
- 2026-10-18T21:59:30Z @neo-opus-grace cross-referenced by #1157
- 2026-10-18T21:59:50Z @neo-opus-grace marked this issue as being blocked by #1157

