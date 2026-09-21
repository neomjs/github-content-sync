---
id: 375
title: The fleet registry cannot record that it owns a seat's launches
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-19T13:43:38Z'
updatedAt: '2026-09-19T15:00:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/375'
author: neo-opus-grace
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
blocking:
  - '[x] 171 No seat can get its first launch from the cockpit'
closedAt: '2026-09-19T15:00:51Z'
---
# The fleet registry cannot record that it owns a seat's launches

## Context

Blocking leaf of neomjs/neo-agent-institution#171 — *no seat can get its first launch from the cockpit*, the core of the v13.2 FM §04 bar (*"the operator starts an agent from the cockpit UI instead of a terminal"*). The decision on that ticket ([comment 5742353369](https://github.com/neomjs/neo-agent-institution/issues/171#issuecomment-5742353369)) chose **launch ownership as a registry fact**. The alternatives fell to their own falsifiers: presence is an activity proxy, and a Start the server refuses cannot refuse without this fact. This ticket is the Brain half.

## The Problem

`FleetManager#fleetRuntimeStatus` reports a registered seat with no process record as `state: 'unmanaged'`, `confidence: 'none'`. That is correct for a seat the fleet never launched: never-launched is not stopped, and an external harness may be running right now. `fleetCockpitStatus` then maps `unmanaged` to runtime `not-wired`, and the cockpit's Start stays disabled.

A seat the operator creates **in the cockpit** to be launched **by the fleet** gets the same verdict, because the registry holds nothing that distinguishes it. So its first launch cannot come from the cockpit either.

## The Architectural Reality

- `FleetRegistryService` rows are `{id, githubUsername, harnessType, modelProvider, mcpServers, mcpTarget, metadata, createdAt, updatedAt}`. `metadata` is free-form, and `updateAgent` merges into it. The one authority-bearing key, `metadata.launch`, is already guarded: it is writable only through `setLaunchOverride`.
- `defineAgent` has two callers. `FleetControlBridge#defineAgent` serves the cockpit's `AddAgentFlow`. `ai/scripts/fleet/onboardPeer.mjs:743` is Phase A of the team's onboarding conductor, and it runs *before* the roster ceremony that its own Phase B preflight enforces.
- The facet-verb precedent is `FleetManager#setRepo` and `#setAvatar`, each exposed on `FleetControlBridge` and listed in `src/fleet/contract/wire.mjs` `FLEET_WIRE_METHODS`.
- `FleetLifecycleService#start` has one liveness check, `isRunning(id)`, which reads the fleet's own record. It cannot see an external session, which is why the fact has to exist before any cockpit Start is offered.

## The Fix

1. **`launchOwner: 'fleet' | 'external'`**, a first-class registry field, not `metadata`. An absent value reads as `external`: every existing row, and `onboardPeer`'s Phase A.
2. **`defineAgent`** accepts `launchOwner` in its curated intent, as either value of the enum. Only the cockpit's define intent sends `fleet`. A value outside `fleet` / `external` is rejected, and absence means `external`.
3. **`FleetManager#adoptAgent(id)`** and **`#releaseAgent(id)`** are the only writes after birth. They are facet verbs beside `setRepo`, exposed on `FleetControlBridge` and in `FLEET_WIRE_METHODS`, and each records `launchOwnerSince`. `updateAgent`, `configureAgent` and `setAvatar` cannot change the field.
4. **`fleetRuntimeStatus`**: no process record and `launchOwner === 'fleet'` → `state: 'stopped'`, `confidence: 'inferred'`, `reason: 'fleet-launched seat with no process record'`. External rows keep `unmanaged` / `none` unchanged. The method's docblock carries the refined rule: never-launched is not stopped, except where the fleet is the only sanctioned launcher, and the confidence says it is a policy inference.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| registry row `launchOwner` | `FleetRegistryService` definition shape | `'fleet' \| 'external'`; absent reads as `external` | legacy rows unchanged | the class docblock's definition shape | unit: legacy row and field-less define → `external` |
| `defineAgent` intent | `FleetRegistryService#defineAgent` | accepts `launchOwner: 'fleet' \| 'external'`; rejects values outside that two-value enum | omitted → `external` | method docblock | unit: `fleet` accepted, `'banana'` rejected |
| `adoptAgent` / `releaseAgent` | new, beside `FleetManager#setRepo` | set `fleet` / `external`, record `launchOwnerSince`; unknown id → `null` | — | method docblocks; `wire.mjs` | unit + bridge dispatch arm |
| `updateAgent` / `configureAgent` / `setAvatar` | existing | cannot write `launchOwner` | — | — | unit: patch attempt rejected or ignored, field unchanged |
| `fleetRuntimeStatus` row | `FleetManager#fleetRuntimeStatus` | fleet-owned + no record → `stopped` / `inferred` + reason | external → `unmanaged` / `none` (unchanged) | method docblock | unit: both rows side by side |

## Acceptance Criteria

- [ ] A field-less definition — every legacy row, and `onboardPeer`'s Phase A shape — reads `launchOwner: 'external'` and stays `unmanaged` / `none` in `fleetRuntimeStatus`.
- [ ] `defineAgent({…, launchOwner: 'fleet'})` persists the fact. A never-launched fleet-owned seat reads `stopped` / `inferred` with its reason, and `fleetCockpitStatus` treats it as supervised.
- [ ] `adoptAgent` / `releaseAgent` flip the fact, record `launchOwnerSince`, and are callable through `FleetControlBridge` and the wire list. No other write surface changes the field.
- [ ] Once a process record exists, the observed state wins over the inference — for a fleet-owned seat, `running` after a start and `stopped` after a stop, both `observed`.
- [ ] Red-first: the `stopped` / `inferred` arm fails on `dev`, and a mutation that defaults `defineAgent` to `fleet` turns the legacy/`onboardPeer` arm red.

## Out of Scope

- The cockpit side: the define intent, the card's disabled reason and Adopt control, the §04 e2e arms — neomjs/neo-agent-institution#171.
- Per-owner scoping of these writes (`ownerPrincipal`, `CAN_ADMINISTER_FLEET_OF`) — neomjs/neo-agent-brain#83 applies it to these verbs exactly as it does to start and stop.
- The identity a cockpit-added seat boots with — tenant identity, brain#83.
- Widening `running` to a tri-state; the docblock already names that residual and its sole consumer.

## Avoided Traps

- **A `defineAgent` default of `fleet`.** It would arm the cockpit's Start between `onboardPeer`'s phases, bypassing the roster preflight.
- **`metadata.launchOwner`.** A free-form merge surface must not carry an authority fact that enables a Brain-credentialed spawn — the same reason `metadata.launch` has a stop-line.
- **`stopped` for every never-launched seat.** That is the invented verdict the method's docblock exists to refuse.

## Related

neomjs/neo-agent-institution#171 (parent outcome) · neomjs/neo-agent-brain#83 · neomjs/neo-agent-brain#28 (bench/unbench, the same class of recorded operator decision)

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-09-19T13:42:25Z; no equivalent. Keyword searches "launch owner", "adopt agent", "unmanaged": only #28 (adjacent).
- A2A: my own lane-intent (13:37Z) and lane-claim (13:42Z) on the parent; no competing claim.
- Memory Core: the parent's filing session recorded the verified map; no prior decision against a registry fact.
- Structure map (`npm run ai:structure-map -- --files --loc`): owning folder `ai/services/fleet/`; facet-verb precedent `FleetManager#setRepo` / `#setAvatar`.

Decision Record impact: none — aligned with brain#83's later per-owner scoping. No AiConfig touch.

Origin Session ID: bd178faf-66b1-4f57-8c53-8224751e92c2

Retrieval Hint: "fleet launch ownership registry fact adopt release unmanaged stopped inferred first launch cockpit"


## Timeline

- 2026-09-19T13:43:38Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-19T13:43:39Z @neo-opus-grace added the `enhancement` label
- 2026-09-19T13:43:40Z @neo-opus-grace added the `ai` label
- 2026-09-19T13:43:40Z @neo-opus-grace added the `agent-os` label
- 2026-09-19T13:53:13Z @neo-opus-grace cross-referenced by PR #377
- 2026-09-19T13:55:06Z @neo-opus-grace referenced in commit `92bbff8` - "test(fleet): an inferred stop reaches the cockpit as one lifecycle/runtime pair (#375)

The cockpit marks the runtime invalid when the lifecycle and the runtime source disagree on source or confidence, so the pairing fleetCockpitStatus produces by construction is pinned where the Institution depends on it."
- 2026-09-19T15:00:51Z @tobiu referenced in commit `8e09275` - "Merge pull request #377 from neomjs/grace/375-launch-owner

feat(fleet): the registry records who launches a seat, so the cockpit can start a fleet-launched one the first time (#375)"
- 2026-09-19T15:00:51Z @tobiu closed this issue
- 2026-09-19T16:43:40Z @neo-opus-grace cross-referenced by #382
- 2026-09-19T18:50:20Z @neo-gpt-emmy cross-referenced by PR #383

