---
id: 1044
title: Place lmsReplacementFailuresPath in buildPackagedBrainEnv
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-23T15:44:35Z'
updatedAt: '2026-11-23T16:21:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1044'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-23T16:21:02Z'
---
# Place lmsReplacementFailuresPath in buildPackagedBrainEnv

## Context

After `neomjs/neo-agent-brain` PR `neomjs/neo-agent-brain#1910` (`2f4c3d1c4e`, resolving `neomjs/neo-agent-brain#493`) added the Tier-1 plane member `orchestrator.recoveryActuator.lmsReplacementFailuresPath` (`NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH`, `{planeMember: true}`), `neomjs/neo-agent-institution`'s `Explicit Brain contract` CI job fails across all four config bases (`tier-1`, `memory-core`, `knowledge-base`, `neural-link`), blocking outside contributor PR `#1041` (`thib2306681-spec`) and Dependabot PRs `#1042` / `#1043`:

```text
planeConfig.assertPlaneMemberCoherence: plane.dataRoot resolves to "<workDir>" but 1 claimed member(s) still derive from the build-time anchor (orchestrator.recoveryActuator.lmsReplacementFailuresPath) — relocating the plane requires per-member placement (member env bindings / the per-profile election); a partially-moved plane fails closed.
```

Curated labels (`good first issue`, `help wanted`) are omitted: urgent cross-repo CI regression owned and resolved immediately by `@neo-gemini-maryam`.

## The Problem

In `neomjs/neo-agent-brain` (`ai/configBase.mjs:2493-2495`), `orchestrator.recoveryActuator` declares three `planeMember: true` paths:
- `healAttemptsPath` (`NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH`)
- `lmsReplacementFailuresPath` (`NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH`)
- `recoveryRunStateDir` (`NEO_RECOVERY_ACTUATOR_RUN_STATE_DIR`)

`harness/brain.mjs#buildPackagedBrainEnv` (`harness/brain.mjs:952-953`) relocates `NEO_PLANE_DATA_ROOT` to `dataRoot` (`<userData>/brain`) and places `NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH` and `NEO_RECOVERY_ACTUATOR_RUN_STATE_DIR` under `path.join(dataRoot, 'orchestrator', ...)`, but lacks `NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH`. Because `assertPlaneMemberCoherence` fails closed when any declared `PLANE_MEMBER_PATHS` entry remains on its build-time anchor default, both packaged own-mode boot against current Brain `dev` and `test/playwright/unit/harness/brain.spec.mjs:656` fail.

## The Architectural Reality

- `harness/brain.mjs#buildPackagedBrainEnv` (`harness/brain.mjs:927-1000`) is the single packaged product profile consumed by `harness/main.mjs`, `harness/seatRenameBroker.mjs`, `harness/seatRootBroker.mjs`, `test/playwright/unit/harness/brain.spec.mjs`, and `test/playwright/unit/harness/pack.spec.mjs`.
- Per `#347` (`PR #348`), recovery-actuator and route-attribution plane members nest inside `path.join(dataRoot, 'orchestrator', ...)` in the packaged profile (matching `NEO_AI_ORCHESTRATOR_DIR`).
- Structure map check (`npm run ai:structure-map -- --files --loc` in `neomjs/neo-agent-brain`): Tier-1 leaf authority lives in `ai/configBase.mjs` (`633` code LOC); Institution's profile placement lives in `harness/brain.mjs`.
- Design authority: `"Relocated to the per-user root <userData>/brain (NEO_PLANE_DATA_ROOT, harness/main.mjs:979), with every PLANE_MEMBER_PATHS member placed beneath it, so §10.4/§10.5's member check fails a member left on the bundle's build-time default."` (`neomjs/neo-agent-brain` `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` §10.7).

## The Fix

Add `NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH: path.join(dataRoot, 'orchestrator', 'lms-replacement-failures.json')` in `harness/brain.mjs#buildPackagedBrainEnv` alongside `NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH` and `NEO_RECOVERY_ACTUATOR_RUN_STATE_DIR`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `buildPackagedBrainEnv` (`harness/brain.mjs:927`) | `neomjs/neo-agent-brain` `ADR 0019` §10.5 & §10.7; `harness/brain.mjs:904-907` | Serializes `NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH` to `path.join(dataRoot, 'orchestrator', 'lms-replacement-failures.json')` | Fail-closed via `assertPlaneMemberCoherence` if omitted | Inline JSDoc on `buildPackagedBrainEnv` unchanged (already covers all `PLANE_MEMBER_PATHS`) | `test/playwright/unit/harness/brain.spec.mjs` & `test/playwright/unit/harness/pack.spec.mjs` under locked Brain and `NEO_AGENTOS_RUNTIME_ROOT` (`dev`) |

## Decision Record impact

`aligned-with neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot` (§10.5 & §10.7 explicit per-member packaged-profile placement).

## Acceptance Criteria

- [ ] AC-1: `buildPackagedBrainEnv` (`harness/brain.mjs`) binds `NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH` to `path.join(dataRoot, 'orchestrator', 'lms-replacement-failures.json')`, satisfying `pack.spec.mjs` userData/plane-root containment checks.
- [ ] AC-2: `test/playwright/unit/harness/brain.spec.mjs` passes both under locked `node_modules/neo-agent-brain` and with `NEO_AGENTOS_RUNTIME_ROOT` pointing at `neomjs/neo-agent-brain` `dev` (`>= 2f4c3d1c4e`), with all four config bases (`tier-1`, `memory-core`, `knowledge-base`, `neural-link`) passing `assertPlaneMemberCoherence`.

## Out of Scope

- Changing `buildBrainProfile` (`UNIT_TEST_MODE: '1'`), which does not relocate `NEO_PLANE_DATA_ROOT`.
- Bumping the pinned `neo-agent-brain` commit hash in `package.json` (`resolve-org-dev` already tests `dev` in both `Isolated Institution` and `Explicit Brain contract`).

## Avoided Traps

- **Deriving member paths dynamically at runtime inside `buildPackagedBrainEnv`:** `ADR 0019` §10.5 requires explicit per-member profile placement rather than implicit runtime rebasing, and `buildPackagedBrainEnv` is synchronous process-env serialization before any Brain child boots.
- **Nesting under `orchestrator-daemon/` instead of `orchestrator/`:** `buildPackagedBrainEnv` places `NEO_AI_ORCHESTRATOR_DIR`, `NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH`, and `NEO_RECOVERY_ACTUATOR_RUN_STATE_DIR` under `path.join(dataRoot, 'orchestrator', ...)` (`harness/brain.mjs:939-953`).

## Related

- `#347` · `#424` · `neomjs/neo-agent-brain#493` · `neomjs/neo-agent-brain#1910` · `#1041`

Live latest-open + closed sweep: checked latest 25 open (`59` open total — red band; `#347` is closed and `#424` is the Row-5 parent epic, so a one-PR regression leaf is required) + open and closed search (`lmsReplacementFailuresPath OR buildPackagedBrainEnv OR planeMember`, `Out of Scope` / `Avoided Traps` read on `#347`) at `2026-11-23T15:43Z`; no open equivalent or prior exclusion found.
A2A claim sweep: checked latest 30 messages (`status: 'all'`) at `2026-11-23T15:43Z`; `@neo-gpt-sophie` (`MESSAGE:619ca36a`) handed triage to `@neo-gemini-maryam`, no competing `[lane-claim]`.
MC sweep: `"buildPackagedBrainEnv NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH explicit contract"` + `"Institution Explicit Brain contract fails plane member build anchor"`, 10 results, no prior decision against this placement found.
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-institution`.

Origin Session ID: e8124dc1-2dcf-4cfb-b94a-3f0c4c23c560
Retrieval Hint: "buildPackagedBrainEnv NEO_RECOVERY_ACTUATOR_LMS_REPLACEMENT_FAILURES_PATH Explicit Brain contract assertPlaneMemberCoherence"

## Timeline

- 2026-11-23T15:44:35Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-23T15:44:36Z @neo-gemini-maryam added the `bug` label
- 2026-11-23T15:44:36Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-23T15:44:37Z @neo-gemini-maryam added the `ai` label
- 2026-11-23T15:44:37Z @neo-gemini-maryam added the `regression` label
- 2026-11-23T15:45:51Z @neo-gemini-maryam added parent issue #424
- 2026-11-23T15:50:23Z @neo-gemini-maryam cross-referenced by PR #1045
- 2026-11-23T16:21:02Z @tobiu referenced in commit `226c050` - "fix(brain): place lmsReplacementFailuresPath in packaged Brain env (#1044) (#1045)"
- 2026-11-23T16:21:03Z @tobiu closed this issue

