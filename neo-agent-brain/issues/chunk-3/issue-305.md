---
id: 305
title: LM Studio backend crashes cause unbounded residency retries
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-04T10:31:34Z'
updatedAt: '2026-09-04T12:56:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/305'
author: neo-gpt-emmy
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
closedAt: '2026-09-04T12:56:37Z'
---
# LM Studio backend crashes cause unbounded residency retries

## Context

On 2026-09-04 the machine-local Agent OS lost its LM Studio chat lane on every synthesis attempt. The host-edge supervisor stayed alive and kept trying to restore `google/gemma-4-26b-a4b`, but LM Studio 0.4.21+2 had selected `mlx-llm-mac-arm64-apple-metal-nax-advsimd@1.11.0`, whose load child referenced a missing `app-mlx-generate-mac26-arm64@33` vendor pack.

This is measured as two related but distinct observations:

- **Backend crash class:** 55 macOS crash reports carry `SIGABRT` in `llm_engine::MLXAmphibianEngine::load` plus `failed to get the Python codec of the filesystem encoding`. The selected runtime referenced vendor pack `@33`; installed packs stopped at `@32`.
- **Neo retry amplification:** the host-edge log emitted degraded LM Studio readiness continuously from 08:43Z through 09:56Z. Each missing-model pass attempted another additive load, so a deterministic vendor failure became repeated crashing child processes and a user-visible `Model is unloaded` lane.

Selecting installed MLX runtime `1.10.1` was the reversible recovery. It stopped this SIGABRT class—zero matching crash reports after the switch—and later real Knowledge Base syntheses succeeded with Gemma still resident at context 262144, parallel 1, and `ttlMs:null`. One earlier post-switch synthesis still returned an empty answer with `ExplicitModelUnloadError`, so this ticket does **not** claim the wider residency incident is fully explained or that #29 is closed.

LM Studio's own documentation says models loaded with `lms load` have no TTL by default and remain until manually unloaded: https://lmstudio.ai/docs/developer/core/ttl-and-auto-evict. Its public bug tracker has the same MLX/Python-codec failure class under a different app/runtime release: https://github.com/lmstudio-ai/lmstudio-bug-tracker/issues/1189. That is corroboration of the vendor failure shape, not proof about Neo's retry policy.

## The Problem

A deterministic `lms load` failure has no terminal disposition in the current residency loop.

At Brain `dev@80c551e872`:

- `ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs:159-180` calls `ensureLmsModelsLoaded()` with `allowPartial:true`.
- `ai/services/graph/providerReadinessHelper.mjs:1815-1875` records a failed load, logs it, and returns degraded readiness.
- `ai/daemons/orchestrator/services/ProcessSupervisorService.mjs:573-619` runs that hook after each liveness confirmation and records only the generic degraded/failed outcome in its visible log.
- The provider-readiness path does not use the existing `createBoundedRetryGate` primitive and carries no equivalent same-failure circuit.

That shape is correct for a transient missing model, but wrong for a runtime that aborts the same child on every attempt. Nothing distinguishes “try again” from “the selected backend cannot start this model until external runtime state changes.” The retry creates cost without increasing the chance of recovery, and the actionable vendor/runtime identity is visible only in LM Studio's own logs.

The merged identifier-collision repair (#265 / PR #264) is adjacent, not duplicative. It adopts an already-resident sufficient instance when `lms load` reports “identifier already exists.” This incident has no resident instance to adopt; the selected MLX backend aborts during load.

## The Architectural Reality

- The graphless host edge owns LM Studio supervision and no container-plane work. ADR 0019 §10.7 and `src/composition/orchestrator/hostEdgeProfile.mjs` elect that one lane.
- `providerReadinessHelper.mjs` owns LM Studio-specific residency observation and additive load. A provider-specific failure circuit belongs at that boundary; `ProcessSupervisorService` remains provider-agnostic.
- Runtime selection is operator/vendor state. Neo may observe and report it, but must not silently downgrade or switch installed runtimes.
- Collision adoption, transient retries, and deterministic backend failure are separate outcomes. A circuit for the third must not weaken the first two.
- The currently deployed host edge is the pre-cut runtime `03035d1b2a`; #253 owns the Brain-built-image/runtime cutover. A merged Brain fix is not a live-host fix until deployment identity proves it is running.

Structure-map result: `ai/services/graph/providerReadinessHelper.mjs` is the existing owner and `ai/daemons/orchestrator/services/ConfiguredTaskDefinitionsService.mjs` / `ProcessSupervisorService.mjs` are the existing composition surfaces. No new `.mjs` file is required.

## The Fix

1. Classify LM Studio load failure as structured evidence: model id, operation, error code/signal, a sanitized stable failure fingerprint, and an observable selected-runtime fingerprint when available.
2. Bound identical load failures. After a small documented number of equivalent failures with unchanged host, configured model/load shape, and sanitized failure fingerprint, open a provider-specific circuit and stop spawning more load children.
3. Surface the circuit through the existing readiness/task-outcome envelope: why it opened, what runtime was observed, when/what can re-arm it, and the last exact load failure. The normal host-edge log must carry the actionable reason instead of only “degraded readiness.”
4. Re-arm on evidence: the model becomes resident; host/model/load shape changes; or an **observed** runtime selection changes from known A to known B. The vendor CLI has no structured runtime-list mode, so `unknown` never counts as change. A fixed cooldown admits one half-open probe; process restart is the explicit reset.
5. Preserve the current collision-adoption and transient-retry paths. Do not restart a live LM Studio server solely because one model backend is incompatible.
6. Keep runtime selection operator-owned. Diagnostics may name `lms runtime select` as the recovery surface, but production code must not choose a fallback version.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ensureLmsModelsLoaded()` result | `providerReadinessHelper.mjs`; #29/#265 collision taxonomy | Distinguish transient failure, identifier adoption, and deterministic repeated load failure; return structured failure/circuit evidence | Unknown failures remain degraded and bounded; never infer success | JSDoc at the helper/result fields | Unit sequence: same backend failure repeats, circuit opens, later calls do not invoke `loadModel` |
| LM Studio failure fingerprint | this ticket; actual `lms load` settlement | Stable across equivalent sanitized failures and keyed by host + model/load shape; runtime rows are best-effort diagnostics | `lms runtime ls` has no JSON mode: preserve prior known identity across `unknown`, and use resident/config change or a half-open probe to recover | classifier JSDoc | Positive/negative controls: same failure accumulates; changed host/model/shape or failure resets; observed A→B re-arms |
| Host-edge readiness/task outcome | `ConfiguredTaskDefinitionsService` + `ProcessSupervisorService` | Report `load-blocked` / equivalent terminal-degraded state with cause and re-arm condition | Other host-edge tasks continue; no LM Studio server restart from this condition | operator-facing log + task-state envelope | Exact outcome/log assertion, including no repeated load call |
| Recovery/re-arm | host-edge LM Studio residency owner | Re-probe after observed runtime/model/residency change or explicit reset; successful residency clears the circuit | One bounded cooldown probe may remain | JSDoc | Runtime-fingerprint mutation and resident-model controls |
| Deployment truth | #253 | A Brain source fix is reported separately from the deployed host-edge revision | Pre-cut host keeps the manual 1.10.1 workaround | PR evidence + deployment receipt | live host reports candidate Brain revision, two real syntheses, resident model afterward |

Decision Record impact: `aligned-with ADR 0019 §10.7` — this hardens the host edge's elected LM Studio lane without changing task authority or AiConfig resolution. If the implementation adds retry/circuit leaves, they are declared once in AiConfig and read at the use site under ADR 0019; no shadow defaults or env re-reads.

## Acceptance Criteria

- [ ] A seeded deterministic `lms load` backend failure opens a circuit after a documented finite number of equivalent attempts; subsequent unchanged readiness cycles invoke no additional load child.
- [ ] The circuit is mutation-sensitive: changed host/model/load shape or failure fingerprint resets the streak; sufficient residency clears it; observed runtime A→B re-arms. An observed→unknown transition stays blocked and records `unknown`; the fixed cooldown admits one half-open probe.
- [ ] A successful resident observation clears the circuit, and the existing #265 identifier-collision adoption path remains green.
- [ ] A transient one-off load failure can recover on the next admitted attempt; the fix does not turn every load error into a permanent block.
- [ ] The task outcome and normal host-edge log expose model, selected runtime (or explicit `unknown`), terminal retry disposition, last sanitized error, and re-arm condition. The operator does not need a vendor-log archaeology pass to know why retries stopped.
- [ ] The deterministic failure arm proves the LM Studio server is not restarted and an already-resident other role/model is not unloaded.
- [ ] Retry bounds use a separately justified fixed-cadence finite mechanism: the generic gate is not reused because it chooses its key before the post-load error exists and would count pre-adoption outcomes. Any new policy values are AiConfig leaves with no hidden duplicate, per ADR 0019.
- [ ] Exact-current-head unit tests cover the classifier, bound, re-arm, and collision/transient controls; removing the circuit or collapsing deterministic failure back into generic degraded readiness makes a named test fail.
- [ ] **[L3-deferred — deployment handoff]** after #253 or an equivalent host-edge cutover, the live host runs the candidate Brain revision; two separated real syntheses succeed and `lms ps --json` still shows Gemma resident afterward.
- [ ] The incident record keeps the recovery bound honest: MLX 1.10.1 stopped the observed SIGABRT class, but the one post-switch `ExplicitModelUnloadError` remains related to #29 until independently closed.

## Out of Scope

- Automatically selecting or downgrading an LM Studio runtime.
- Repairing LM Studio's missing vendor pack or upstream MLX backend.
- Closing #29's TTL/JIT and sustained-absence residuals.
- Reworking container-plane model providers.
- Deploying Brain source onto the pre-cut host; #253 owns that cutover.
- Treating the stale Knowledge Base corpus as duplicate or architectural evidence; live GitHub/source decide this ticket.

## Avoided Traps

- **“Retry harder.”** Identical deterministic process aborts are not increasing-probability retries.
- **Generic supervisor circuit.** The discriminating evidence is LM Studio-specific; making every degraded readiness hook share this policy would couple unrelated tasks.
- **Silent runtime downgrade.** Vendor runtime choice is operator-owned external state.
- **Calling `ttlMs:null` proof of total recovery.** It proves the current resident was loaded without TTL, not that no request can still interrupt it.
- **Conflating #29.** Identifier collision and backend process abort have different positive evidence and different terminal actions.
- **Claiming merged means deployed.** The host currently executes a pre-cut Engine snapshot.

## Related

- #29 — residency-loop incident and remaining TTL/JIT/sustained-absence questions
- #265 / PR #264 — identifier-collision adoption, already merged
- #253 — Brain-built local Agent OS cutover
- LM Studio bug tracker #1189 — same public MLX/Python-codec failure class, different version/context

Live latest-open sweep: checked the 20 newest open Brain issues at 2026-09-04T10:31Z; none covers deterministic LM Studio backend crashes. Full-state `"LM Studio"` search found #29, #265, #131, #84, and #64; none owns this mechanism. A2A all-read-state sweep over the newest 60 messages found only this session's defect/maintenance notices and no competing claim. Memory Core rationale sweep surfaced earlier LM Studio outage/collision history but no missing-vendor-pack or deterministic backend-crash disposition. The Knowledge Base semantic result was excluded because multi-repository ingestion/polling freshness is not established. Own-assignment sweep found no same-surface lane.

Origin Session ID: a518c504-b9ec-40ce-951e-446ce4574fc1

Retrieval Hint: "LM Studio MLX backend load abort Python codec missing vendor pack unbounded residency retry circuit"

Retrieval Hint: host-edge `03035d1b2a`, Brain `dev@80c551e872`, 2026-09-04 08:43Z–10:14Z


## Timeline

- 2026-09-04T10:31:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-04T10:31:36Z @neo-gpt-emmy added the `bug` label
- 2026-09-04T10:31:37Z @neo-gpt-emmy added the `ai` label
- 2026-09-04T10:31:37Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-04T11:53:33Z @tobiu cross-referenced by PR #308
- 2026-09-04T12:35:00Z @tobiu referenced in commit `d95110d` - "feat(orchestrator): bound LM Studio load failures (#305)"
- 2026-09-04T12:35:00Z @tobiu referenced in commit `8218a1f` - "fix(orchestrator): reserve half-open cooldown on admission (#305)"
- 2026-09-04T12:56:37Z @tobiu referenced in commit `25d374e` - "Merge pull request #308 from neomjs/codex/305-lms-load-failure-circuit

feat(orchestrator): bound LM Studio backend load failures (#305)"
- 2026-09-04T12:56:37Z @tobiu closed this issue

