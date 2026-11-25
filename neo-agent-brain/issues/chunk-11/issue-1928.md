---
id: 1928
title: Route at-cap store memory saturation to record instead of raise-ceiling
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-25T17:03:48Z'
updatedAt: '2026-11-25T18:11:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1928'
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
blockedBy: []
blocking: []
closedAt: '2026-11-25T18:11:02Z'
---
# Route at-cap store memory saturation to record instead of raise-ceiling

## Context

Split from #1843 (Fix 1) per `@neo-opus-ada`'s intake (`https://github.com/neomjs/neo-agent-brain/issues/1843#issuecomment-7168633243`).

Live `inspect_deployment` diagnostics on `2026-11-18` showed `chroma` (`chromadb/chroma:1.5.9`) running at `16.99 GB / 17.18 GB` (`88.89%` of its `16 GiB` cgroup limit — already at `CONTAINER_MEMORY_CEILING_MAX_BYTES = 16 * 1024 ** 3`). Because `88.89% >= storeMemorySaturationPercent` (`80%`), `ContainerHealthDiagnosisService` emits `actionClass: 'raise-ceiling'` (`reason: 'store-ceiling-exhaustion'`) on every diagnosis cadence, and `RecoveryActuatorService#raiseComposeServiceCeiling` refuses the proposed `32 GiB` step with `status: 'failed'`, `reasonCode: 'executor-failed'` (`72` failed self-heal ledger events).

## The Problem

`RecoveryActuatorService.spec.mjs:2028-2052` (`"the ratchet terminates: a raise past the registry cap is a recorded refusal, and no live write happens"`) intentionally enforces that if `raise-ceiling` is called when `liveLimitBytes >= 16 GiB`, the actuator refuses with `executor-failed` — assuming the cadence envelope would stop retrying. However, as documented in `ContainerHealthControllerService.mjs:34-41`, `getCurrentAttemptState` resets `attemptCount` every `maxAttemptsWindowMs`, so dispatching `raise-ceiling` for an already-at-cap store causes steady per-window `executor-failed` churn until the futility breaker freezes it.

This is the exact store-memory counterpart of closed #1756 (`prune-build-cache` routed to `record` in `ContainerHealthDiagnosisService` once reclaimable cache is already at the warm floor).

## The Architectural Reality

- `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs`:
  - `collectStatsFacts` (lines 946–1134) computes sustained `memorySaturation` facts for store-backed services (`serviceClass: 'store'`).
  - `classifyService` (lines 1781–1795) unconditionally returns `actionClass: CONTAINER_HEALTH_ACTION_CLASSES.raiseCeiling`, `reason: 'store-ceiling-exhaustion'` whenever `storeMemoryFacts.length > 0`, without checking whether the store's live `memoryLimitBytes` is already `>=` the `container-memory-ceiling` knob's declared `max` (`ai/services/memory-core/helpers/recoveryKnobRegistry.mjs:56,124`).
- `ai/daemons/orchestrator/services/ContainerHealthControllerService.mjs`: routes `CONTAINER_HEALTH_ACTION_CLASSES.record` through `recordWithoutAction` → `recoveryActuator.recordDiagnosis` (enveloped by `decideFutilityFreeze`, zero `executor-failed` actuator throws).
- `test/playwright/unit/ai/daemons/orchestrator/services/RecoveryActuatorService.spec.mjs:2028-2052`: preserves the actuator's defense-in-depth `executor-failed` refusal if a caller ever invokes `apply('chroma', 'raise-ceiling')` directly at `16 GiB`.

Design authority: `ai/services/memory-core/helpers/recoveryKnobRegistry.mjs:47-52` ("At ~34% of the 47 GiB Docker VM the sizing was measured against, a store that still saturates has outgrown ceiling-raising as a heal: the next move is a corpus-architecture decision ... The cap is where autonomy ENDS on purpose") and #1756 diagnosis-side `record` routing precedent.

## The Fix

1. In `ContainerHealthDiagnosisService#collectStatsFacts`, attach the observed live container `memoryLimitBytes` (from `inspect.HostConfig.Memory` when `proved`, falling back to uniform positive `memory_stats.limit` across `samples`) to `memorySaturation` `details` for `container`-scoped (`!isHeapScope`) facts.
2. In `ContainerHealthDiagnosisService#classifyService` (`storeMemoryFacts` branch), look up the matching ceiling knob's `max` (`RECOVERY_KNOBS['container-memory-ceiling']` when `serviceKey === 'chroma'` / matching `serviceKey` ceiling knob): when `memoryLimitBytes` is finite and `>= max`, classify as `actionClass: CONTAINER_HEALTH_ACTION_CLASSES.record`, `reason: 'store-ceiling-at-cap'` (carrying `memoryLimitBytes` and `ceilingMaxBytes` in evidence) instead of `raise-ceiling`. Leave `RecoveryActuatorService#raiseComposeServiceCeiling`'s defense-in-depth refusal (`RecoveryActuatorService.spec.mjs:2028`) intact.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ContainerHealthDiagnosisService` store memory classification (`ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs`) | `recoveryKnobRegistry.mjs:47-52,124` + #1756 `record` precedent + ADR-0025 / ADR-0026 | Store with sustained `memorySaturation` below knob `max` (`< 16 GiB`) emits `actionClass: 'raise-ceiling'` (`store-ceiling-exhaustion`); at or above `max` (`>= 16 GiB`) emits `actionClass: 'record'` (`store-ceiling-at-cap`) | Unknown/missing `memoryLimitBytes` keeps `raise-ceiling` so actuator `inspect` decides | JSDoc on `collectStatsFacts` & `classifyService` | `ContainerHealthDiagnosisService.spec.mjs` + `ContainerHealthControllerService.spec.mjs` |

Decision Record impact: aligned-with ADR-0019 (reads existing `RECOVERY_KNOBS` registry SSOT; adds no duplicate cap constant), ADR-0025, and ADR-0026 (no new action class).

## Acceptance Criteria

- [ ] `ContainerHealthDiagnosisService` classifies sustained store `memorySaturation` as `actionClass: 'raise-ceiling'` (`reason: 'store-ceiling-exhaustion'`) when `memoryLimitBytes < 16 GiB` (or unknown), and as `actionClass: 'record'` (`reason: 'store-ceiling-at-cap'`) when `memoryLimitBytes >= 16 GiB` (`RECOVERY_KNOBS['container-memory-ceiling'].leaves[0].max`).
- [ ] Unit tests in `ContainerHealthDiagnosisService.spec.mjs` (and controller integration in `ContainerHealthControllerService.spec.mjs`) verify at-cap `chroma` routes to `recorded` with zero `raise-ceiling` `executor-failed` calls, while `RecoveryActuatorService.spec.mjs` remains green.

## Out of Scope

- Widening `CONTAINER_MEMORY_CEILING_MAX_BYTES` past `16 GiB` or Chroma HNSW dead-element compaction (#133, #1432, #1434).
- `mc-server` hot-query optimization (#1843) and `mc-server` CPU/heap headroom sizing.

## Avoided Traps

- Mutating `RecoveryActuatorService#raiseComposeServiceCeiling` to return soft-success at `16 GiB`: violates `RecoveryActuatorService.spec.mjs:2028-2052` ("this class signals action failure by throwing, and a second soft-failure convention would let a caller treat a refusal as success", `RecoveryActuatorService.mjs:1128-1131`).
- Hardcoding `16 * 1024 ** 3` in `ContainerHealthDiagnosisService` instead of reading the ceiling knob's `leaves[0].max` from `RECOVERY_KNOBS`.

## Related

Split from #1843 · Precedent: #60, #73, #1756

---

Live latest-open + closed sweep: checked latest 20 open of 154 (`red` band; mechanical 1-PR split from #1843) + open/closed searches (`raise-ceiling`, `ceiling-at-cap`, `prune-freed-nothing`; Out of Scope & Avoided Traps read on #60 and #1756) at 2026-11-25T16:59Z; no equivalent open ticket.
A2A in-flight sweep: latest 30 all-status messages read at 2026-11-25T17:01Z; split requested by `@neo-opus-ada` (`MESSAGE:0f1a6905-9be4-485b-ae33-f0aa031583c5`), zero competing claims on `ContainerHealthDiagnosisService`.
MC sweep: `"chroma store-ceiling-exhaustion raise-ceiling executor-failed 16 GiB cap"`, 5 results (`2f1b3366`, `086cb2a5`, `5973982f`), no prior ticket on at-cap diagnosis routing.
Own-assignment sweep: 5 open (#1230, #1719, #1853, #1854, #1855, #1927), none overlapping; self-assigning `@neo-gemini-julia` for immediate execution (curated labels omitted because owned immediately).
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs` (existing file, zero new `.mjs` files).

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "chroma store-ceiling-exhaustion 16 GiB cap store-ceiling-at-cap ContainerHealthDiagnosisService record"

## Timeline

- 2026-11-25T17:03:48Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T17:03:50Z @neo-gemini-julia added the `bug` label
- 2026-11-25T17:03:50Z @neo-gemini-julia added the `ai` label
- 2026-11-25T17:03:51Z @neo-gemini-julia added the `architecture` label
- 2026-11-25T17:03:51Z @neo-gemini-julia added the `agent-os` label
- 2026-11-25T17:04:50Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-25T17:09:21Z @neo-gemini-julia cross-referenced by PR #1930
- 2026-11-25T17:57:03Z @neo-gemini-julia referenced in commit `212bbdd` - "fix(orchestrator): require proved inspect cgroup limit and attach ceilingMaxBytes (#1928)"
- 2026-11-25T18:11:02Z @tobiu referenced in commit `4187695` - "fix(orchestrator): route at-cap store memory saturation to record (#1928) (#1930)

* fix(orchestrator): route at-cap store memory saturation to record (#1928)

* fix(orchestrator): require proved inspect cgroup limit and attach ceilingMaxBytes (#1928)"
- 2026-11-25T18:11:04Z @tobiu closed this issue

