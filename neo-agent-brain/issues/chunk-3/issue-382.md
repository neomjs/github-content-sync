---
id: 382
title: An explicit release does not stop the fleet from starting a seat it has run
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-19T16:43:39Z'
updatedAt: '2026-09-19T19:24:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/382'
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
blocking: []
closedAt: '2026-09-19T19:24:30Z'
---
# An explicit release does not stop the fleet from starting a seat it has run

## Context

Found by @neo-gpt-emmy's review of neomjs/neo-agent-institution#174 (review 5256453012). It is an exact-source probe over Brain `8e09275` through `releaseAgent → fleetRuntimeStatus → createFleetCockpitStatus → SourceHealth.mapFleetSessionHealth → FleetStartPlan.partitionFleetStart`:

| Seat | Owner after | Runtime | Start-fleet eligible |
|---|---|---|---|
| never launched, its own harness | `external` | `unmanaged` / `none` | 0 |
| never launched, fleet-owned | `fleet` | `stopped` / `inferred` | 1 |
| the fleet ran it and it stopped, then released | `external` | `stopped` / `observed` | **1** |

## The Problem

`FleetRegistryService` documents `external` as a seat "in a harness the fleet did not start", and says `setLaunchOwner` "hands it back to a harness the fleet does not start". What ships is narrower. `launchOwner` decides only how a seat with **no process record** reads, in `FleetManager#fleetRuntimeStatus`. Once the fleet holds a record, observation wins, and every start path starts the seat: the cockpit's card, its start plan, and `FleetManager.startAgent`.

So a seat released to its own harness, and running there now, can be launched a second time. That is the hazard decision A was built to prevent (neomjs/neo-agent-institution#171, comment 5742353369). neomjs/neo-agent-institution#174 therefore offers adoption alone, and says in its own words that Start stays off only while the fleet has not run the seat.

## The Architectural Reality

- `FleetManager#fleetRuntimeStatus`: a process record is observation and wins over ownership. Keep this. Relabeling observed history as absent is not a fix.
- `FleetLifecycleService#start`: its one liveness check is `isRunning(id)`, and it has no ownership check.
- `defineAgent` defaults to `external`. `ai/scripts/fleet/onboardPeer.mjs` Phase B launches such a seat through `FleetManager.startAgent`, which decision A relies on.
- `setLaunchOwner` records `launchOwnerSince` for an explicit act. A default row has none.
- The cockpit row already carries a Brain-stamped launch fact, `launchable`, which the start plan and the card read with tri-state honesty.

## Options

| Option | Rule | Blast radius | Falsifier |
|---|---|---|---|
| **1: an explicit release is start authority** | The start verb refuses a seat released by an explicit act (`external` with `launchOwnerSince`), with a stable reason: "released to its own harness: adopt it to start it here". The cockpit row carries that fact beside `launchable`. | None for existing rows: `onboardPeer` and legacy rows keep today's behavior. | Fails if a release must also cover a seat that was never adopted. |
| 2: ownership is start authority for every seat | The start verb starts only `fleet` seats. `onboardPeer` adopts before its Phase B launch. | Every fleet-run `external` row needs an adoption before its next cockpit start. | Fails if a deployment starts default rows from the cockpit today. |
| 3: words only | The contract says what ships: `launchOwner` governs the no-record reading alone. The cockpit never offers release. | None. | Fails if an operator needs to hand a seat back from the cockpit. |

Recommendation: **1**. It keeps observation distinct from permission, honours the operator's recorded act, and changes no existing row. Option 2 is the cleaner rule, if a migration is acceptable.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `launchRefusalOf(definition)` | `src/fleet/contract/launchAuthority.mjs` — moved out of `FleetRegistryService` so the spawn path can ask it without loading AiConfig or a Neo class | `external` + `launchOwnerSince` → the stable reason; anything else → `null` | a row with no ownership act answers `null`, so its process record stays its only start gate | the module's own docblock; exported from the `fleet-contract` barrel | unit: `FleetRegistryService.spec` release/adopt arms, unchanged after the move |
| `FleetManager#startAgent` / `#restartAgent` | `assertStartPermitted` | refuse before anything runs; restart refuses **before** the stop, so it never ends half done | legacy + `onboardPeer` rows unchanged | both method docblocks | unit: released seat refused at entry, with no calls |
| the spawn boundary | `startAgentProvisioned#spawnPermitted` | every spawn re-reads the refusal from the registry **at** the spawn; admission at entry does not carry through asynchronous provisioning | a seat adopted back while preparation runs spawns normally | composer docblock (`@throws`) | unit: release-during-preparation refused (`['ensure','release','prepare']`, `start` never called) + the adopted-back control |
| cockpit row `launchRefusal` | `FleetControlBridge#fleetRoster` | `string \| null`, stamped beside `launchable`, from the same predicate the verb reads | additive; `launchable` and the process observations keep their meaning | bridge docblock | unit: roster arm |
| observation | unchanged | a process record the fleet kept is history, never permission | — | — | the ledger row above is the whole rule |

Institution consumption stays outside this leaf (see Out of Scope).

## Acceptance Criteria

- [ ] The contract doc (`FleetRegistryService` class doc, `setLaunchOwner`) states the chosen semantics, and the code delivers exactly that.
- [ ] Options 1 and 2: starting a released seat is refused with a stable reason, and the cockpit row carries the fact. Unit arms cover the three seats above, plus a released seat that the fleet had run, which is refused.
- [ ] Option 3: the doc changes alone.
- [ ] A release published **while provisioning/preparation runs** does not spawn: the refusal is re-read at the spawn boundary, driven through the real composer, with an adopted-back control.

## Out of Scope

- The Institution's release control. It comes back in its own Institution ticket once the plane honours a release.
- Widening `running` to a tri-state (see `fleetRuntimeStatus`'s own residual note).

## Related

neomjs/neo-agent-institution#171, neomjs/neo-agent-institution#174 · #375, #377 (the launch-owner field and its verbs)

Sweeps: the latest 20 open Brain issues at 2026-09-19T16:43Z, plus a search for "release launchOwner". No equivalent found.

Origin Session ID: bd178faf-66b1-4f57-8c53-8224751e92c2


## Timeline

- 2026-09-19T16:43:39Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-19T16:43:41Z @neo-opus-grace added the `bug` label
- 2026-09-19T16:43:41Z @neo-opus-grace added the `ai` label
- 2026-09-19T16:43:41Z @neo-opus-grace added the `agent-os` label
- 2026-09-19T16:44:38Z @neo-opus-grace cross-referenced by PR #174
- 2026-09-19T17:14:34Z @neo-opus-grace cross-referenced by PR #383
- 2026-09-19T19:03:38Z @neo-opus-grace referenced in commit `6657dfb` - "fix(fleet): a release published during preparation stops the spawn it was admitted for (#382)

`FleetManager.startAgent` admits a start, but provisioning and preparation are
asynchronous: a seat released while they ran still reached `start`. Every spawn in
`startAgentProvisioned` now goes through one `spawnPermitted` helper that re-reads
the refusal from the registry AT the spawn, so no await placed above it can reopen
the window.

`launchRefusalOf` moves from `FleetRegistryService` to `src/fleet/contract/launchAuthority.mjs`.
The composer is a pure module with no Neo runtime by contract, and reaching the
service for the predicate pulled AiConfig and `core/Base` into it — the spec proved
it by failing to load. One predicate, one home, three callers.

Red-first: the release arm fails on the unrepaired composer (1 failed / 20 passed);
the adopted-back arm is its control, and spawns."
- 2026-09-19T19:24:30Z @tobiu referenced in commit `2bdcbdb` - "Merge pull request #383 from neomjs/grace/382-release-start-authority

fix(fleet): an explicit release is start authority, so the fleet refuses a released seat and the cockpit row says why (#382)"
- 2026-09-19T19:24:30Z @tobiu closed this issue

