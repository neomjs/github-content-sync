---
id: 493
title: The LM Studio lane replaces a wrong resident itself
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T14:28:46Z'
updatedAt: '2026-09-25T14:54:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/493'
author: neo-opus-vega
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
# The LM Studio lane replaces a wrong resident itself

## Context

@tobiu, 2026-09-25 ~14:25Z, on #480's "the action stays the operator's": Agent OS is self-diagnosing and self-healing; the daemons and the ADRs for both exist, and a ticket that points a repair at the operator is not acceptable. This leaf is that ruling applied to the LM Studio lane, where the pattern started.

The chain: neomjs/neo#17079 (2026-08) removed automatic eviction after an 879-line readiness state machine had ejected configured roles, and set the contract "readiness adds a missing model, never evicts; wrong-shape replacement is an operator action". #460 → PR #462 (mine, 2026-09-24) kept a mis-shaped resident report-only and made `replacement-required` carry the `lms unload` / `lms load` pair for a person to type. #480 (mine, 2026-09-25) copied the same disposition into the runtime-observed mismatch. On 2026-09-24 the price of that shape was measured on this plane: a JIT-loaded embedding resident at the wrong context blocked the chat role from 10:36Z until a human pinned both at 14:27Z, with 126 cancelled loads in between, and the operator line the design relied on was never printed.

## The Problem

`ensureLmsModelsLoaded` (`ai/services/graph/providerReadinessHelper.mjs:1778`) diagnoses a wrong resident correctly and then stops: `reportReplacementRequired` (`:2030`) returns `status: 'replacement-required'` with the warning "LM Studio loaded-model replacement requires an explicit operator action", and `:1728` composes the exact `lms unload … && lms load …` command for a person. The supervisor prints it. Nothing runs it. The lane stays degraded until a human types two commands, on a plane whose LM Studio serves only Agent OS (both roles pinned, JIT off, since 2026-09-24).

ADR-0026 already says what should happen instead: an un-healable class is recorded with its diagnosis, never paged (#14191), and `warm-provider` (the #17012 amendment, AC-13) is the action class for a native role warm under the heavy-maintenance demand boundary. The LM Studio server is a supervised child of the host edge (`ConfiguredTaskDefinitionsService.mjs:118`, the `lms` task with its readiness `postSpawn`), so the replacement is B0-class: the supervisor's in-process cooldown is the loop guard, and no new privilege is needed. The only thing missing is the action.

## The Architectural Reality

- `providerReadinessHelper.mjs`: `loadLmsModel` (`:1221`, `lms load` with `--identifier`, `--context-length`, `--parallel`, injectable `execFileFn`) exists; the unload primitive neomjs/neo#17079 removed does not. `ensureLmsModelsLoaded` (`:1778`) assesses per role since #462 and returns `replacement-required` for a mis-shaped resident before loading anything for that role.
- `ConfiguredTaskDefinitionsService.mjs:118`: the host edge's `lms` task; its `postSpawn` runs the readiness repair, and `ProcessSupervisorService` owns the cooldown.
- ADR-0026 §2.4 / §2.5 (`warm-provider`, the persisted anti-thrash envelope for B1, the in-process cooldown for B0), #14191 (record-with-diagnosis), #17012 (last-owned recheck before every native role warm); ADR-0025 §2.1 (a probe is a signal, never an actuator, so the diagnosis stays where it is and the action is added beside it).
- #480 / PR #488 (Eos): the runtime-observed `MODEL_MISMATCH` on the LM Studio lane is the second producer of the same diagnosis; #480's body is rewritten today so that its failure carries `{requested, served}` for this heal and no operator-addressed text.

## The Fix

1. A `replace-resident` step inside the LM Studio readiness repair (`ensureLmsModelsLoaded`, the repair `warmProviderResidency` awaits through `repairProviderRoleSetResidency`), under `warm-provider`'s demand boundary: for a role whose resident is the pinned identifier at the wrong shape, or a foreign model answering the role's requests, unload that resident (`lms unload <identifier>`, the primitive restored with an injectable `execFileFn`), then the existing `loadLmsModel` at the configured shape, then the existing re-probe. Roles that are merely missing keep today's additive load.
2. Bounded, never a loop: one replacement per role per readiness pass, and the supervisor's cooldown between passes; after N consecutive failed replacements of the same role (a persisted counter beside the existing readiness receipt, so a host-edge restart does not reset it), the result is `heal-exhausted` with the diagnosis, recorded, and the lane reads degraded with that receipt.
3. `replacement-required` retires. The result statuses become `replaced` (with what was unloaded and loaded) or `heal-exhausted` (with why); `operatorDiagnostic.summary` becomes the heal receipt the supervisor already prints. No result carries an imperative addressed to a person.
4. neomjs/neo#17079's rule narrows rather than reverses: a routine readiness pass over a correct resident still touches nothing (its control arm stays), and the replacement acts only on a diagnosed wrong resident of a pinned role. The complexity #17079 removed came from evidence-bound eviction inside readiness; this is one bounded action after a diagnosis readiness already makes.

## The route (producer → transport → actuator)

Euclid's falsifier on #488 (review 5319101046): `git grep -n -E 'model-mismatch|warm-provider' -- ai/daemons/orchestrator` finds `warm-provider` routes and no mismatch consumer. The route this leaf wires uses only existing hops:

1. **Producer:** `DeploymentStateBridgeService` already probes the OpenAI-compatible lane's `GET /v1/models` each snapshot cycle (`providerModelIdentityProbe`, `:1297`) and publishes `providerModelIdentity.state = 'mismatch'` when the configured model is not served (`:1330-1335`); today that state carries a sentence addressed to a person ("Point the lane at a served id, or load the configured model…") and no fact. The readiness repair's `replacement-required` (`providerReadinessHelper.mjs:2030`) is the shape producer (`lms ps` context vs configured). The live `MODEL_MISMATCH` (#480 / #488) is a third observation at the request boundary; it protects the data (no vector stored, no text delivered) and reaches only the memory-core friction map, which the orchestrator does not read, so it is corroboration, not the transport.
2. **Transport:** a `providerResidency` fact with `reasonCode: 'wrong-resident'` (details: `configuredModel`, `observedModel`, source `identity-probe` | `readiness-shape`), emitted from the bridge's `mismatch` state and from the readiness result; `ContainerHealthDiagnosisService.isProviderRoleResidencyRecoverable` (`:1522`, today `missing-required-model` only) admits it, so the diagnosis routes to `warm-provider` exactly as a missing role does.
3. **Actuator:** `RecoveryActuatorService.warmProviderResidency` (`:1177`) → `repairProviderRoleSetResidency` → `ensureLmsModelsLoaded` → the replacement step above → re-probe. The persisted failure counter and `heal-exhausted` sit in the readiness result the actuator records; the bridge's next snapshot reads `match`.

The operator-addressed sentence in the bridge's `mismatch` reason retires with `replacement-required`; the state keeps `configuredModel` / `servedModelIds` as the diagnosis.

**Decision Record impact:** aligned-with ADR 0026 (record-with-diagnosis + autonomous action; `warm-provider`); amends the neomjs/neo#17079 operator-action clause for the diagnosed-wrong-resident case, on the operator's 2026-09-25 ruling (this ticket's Context is the record).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ensureLmsModelsLoaded` result `status` | `providerReadinessHelper.mjs:2030` (`replacement-required`) | `replaced` after a successful unload + load + re-probe; `heal-exhausted` after the cap | `metadata-unknown` unchanged: unreadable metadata still refuses every mutation | JSDoc on the helper | spec arms on the injected recorder (order: unload, load, probe) |
| `operatorDiagnostic.summary` | PR #462 | the heal receipt (role, unloaded, loaded, or the exhaustion reason); no `lms …` imperative | absent when nothing was replaced | same | spec asserts the printed line carries no `lms unload` / `lms load` text |
| persisted per-role failure counter | ADR-0026 §2.5 (anti-thrash survives restart) | increments per failed replacement, resets on `replaced` | missing file → 0 | JSDoc | spec: counter survives a fresh helper instance |
| `lms unload` primitive | `loadLmsModel` (`:1221`) as the sibling | `unloadLmsModel(identifier, {execFileFn, timeoutMs})` | exit non-zero → the replacement fails, counted | JSDoc | spec with the injected `execFileFn` |

## Acceptance Criteria

- [ ] With a pinned role resident at the wrong shape, the readiness repair issues `lms unload <id>` then `lms load … --identifier <id> --context-length <configured>` for that role, re-probes, and returns `replaced`; missing roles in the same pass still load additively (red-first: at dev the same fixture returns `replacement-required` and issues no unload).
- [ ] A load that fails N times for the same role returns `heal-exhausted` with the diagnosis, issues no further unload for that role until a `replaced` resets the counter, and the counter survives a new helper instance.
- [ ] A correct resident is never unloaded (the neomjs/neo#17079 control arm, kept from #462's spec).
- [ ] No result, warning or summary carries an imperative addressed to a person (`lms unload`, `lms load`, "operator action"); the supervisor's degraded line shows the heal receipt instead.
- [ ] A bridge `providerModelIdentity.state = 'mismatch'` and a readiness `replacement-required` each yield a `providerResidency` fact with `reasonCode: 'wrong-resident'` that `isProviderRoleResidencyRecoverable` admits, and the diagnosis dispatches `warm-provider` for the lane's target (spec on the injected actuator recorder); the bridge's `mismatch` reason no longer addresses a person.
- [ ] The specs are on `brain-unit.yml`'s run list.

## Out of Scope

- Other providers: Ollama keeps both roles resident by configuration (`KEEP_ALIVE=-1`, `MAX_LOADED_MODELS=2`), and #461 owns the per-role residency readout.
- The detection half: #480 / PR #488 raise the typed mismatch; this leaf consumes it on the LM Studio lane only.
- LM Studio settings (JIT, `unloadPreviousJITModelOnLoad`): set on this plane on 2026-09-24; a settings drift is #460's family, not this action.

## Related

- #480 / PR #488 (the runtime mismatch, Eos; Euclid's review 5319101046 named the missing route), #460 / PR #462 (the per-role assessment this extends), #461, neomjs/neo#17079, ADR 0025, ADR 0026 (#14191, #17012), #13700 / D#13873 (the June shape-aware preload the eviction removal replaced).
- `DeploymentStateBridgeService.mjs:1297-1335` (the identity probe), `ContainerHealthDiagnosisService.mjs:1156, :1522` (the recoverable residency facts), `RecoveryActuatorService.mjs:1177` (`warm-provider` → `warmProviderResidency`).

Live latest-open sweep: the latest 20 open neo-agent-brain issues at 2026-09-25T14:26:55Z, plus a search across open issues for LM Studio resident / replacement-required / unload / evict; no equivalent. A2A in-flight sweep: the last 30 messages (13:20–14:24Z); Eos's claim is #480, the detect half, no claim on the heal.
MC sweep: `query_raw_memories` on the symptom's nouns (a wrong LM Studio resident, `replacement-required` printed for the operator, eviction removed by neomjs/neo#17079) — six results: the June recovery daemon and PR #13941's shape-aware preload, neomjs/neo#17079's removal, my own 09-24 turn that chose report-only under #17079. The prior decision this amends is named above; no other competing decision.
Own-assignment sweep: 7 open in neo-agent-brain, #461 and #480 adjacent (readout, detection), neither owns the action.
Structure map: run today (exit 0); `ai/services/graph/providerReadinessHelper.mjs` owns the readiness repair and `ai/daemons/orchestrator/services/` the actuators; no new file.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e
Retrieval Hint: "LM Studio lane replaces wrong resident itself replacement-required retired heal-exhausted"


## Timeline

- 2026-09-25T14:28:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T14:28:48Z @neo-opus-vega added the `bug` label
- 2026-09-25T14:28:48Z @neo-opus-vega added the `ai` label
- 2026-09-25T14:28:49Z @neo-opus-vega added the `architecture` label
- 2026-09-25T14:28:49Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T14:30:07Z @neo-opus-vega cross-referenced by #480
- 2026-09-25T14:35:51Z @neo-gpt cross-referenced by PR #488

