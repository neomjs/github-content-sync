---
id: 1138
title: 'A stopped seat recovers as `exited`, which disables its Start'
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-18T17:07:53Z'
updatedAt: '2026-10-18T17:34:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1138'
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
closedAt: '2026-10-18T17:34:43Z'
---
# A stopped seat recovers as `exited`, which disables its Start

## Context

Operator report, 2026-10-18 (analysis relayed by @neo-opus-grace), FM install Institution `2f9056a` / Brain `d6a905dc`: Ada's seat was started (15:50:08Z) and stopped (15:50:34Z) from the roster. Its ▶ stays disabled with *"Runtime not nominal · lifecycle and runtime facts contradict"*. Two FM restarts and a machine reboot did not clear it; every other seat starts.

**Observed** (read-only, 17:01Z):
- The Fleet launch-admission journal (`issuer.json`) holds Ada's `snapshot.seats[]` entry as `state: "exited"`, `ownershipUnresolved: false`, `cleanupUnresolved: false`. Its generation is `revoked`, reason `stop-requested`, 15:50:34Z.
- FM `main.log`: `continuity handoff starting: 0 running seat(s)` (15:51:44Z), then `continuity recovery: previous issuer handed-off …; 1 retained generation(s): neo-opus-ada revoked` at 15:55:58Z and 15:57:47Z.

**Inferred** from source (below): the Stop wrote `stopped`, and the first boot after it rewrote that to `exited`. **Unverified:** whether the ▶ was already disabled in the 70 s between the Stop and the first FM quit. No log line covers that window.

## The Problem

#1051 (#980) made admission custody durable. Each Fleet boot reads the previous issuer's snapshot and rebuilds one lifecycle record per saved seat. A seat whose process is gone and not re-adopted is stamped `exited`, whatever state was saved. Nothing else in `ai/` writes that word. The FM roster's fail-closed mapper does not know it, so the row's runtime source reads `invalid` and the Start plan excludes the seat.

Each boot rebuilds the same record from the snapshot and re-persists it. Only a new Start could replace it, and that is the control the roster disables, so the seat stays dead until the code is fixed.

## The Architectural Reality

At `d6a905dc`:
- **Writer:** `ai/services/fleet/FleetLifecycleService.mjs:2331` (`recoverRuntimeCustodyUnderGuard`), the only `state: 'exited'` write.
- **The Stop path writes `stopped`:** `finalizeExitedProcess`, for a record without `electronProfile`. Only Codex Desktop sets one (`deriveHarnessLaunchSpec.mjs:299`).
- **Readers that already treat `exited` as `stopped`:** `runtimeGenerationOf` (:1664, pin `released`) and `handoffRuntimeCustody` (:2393, skipped).
- **Codec:** `seatRuntimeRecord.mjs:154` accepts `exited` on read, and the installed journal holds it.
- **Pass-through:** `fleetCockpitStatus.mjs` copies `runtime.state` into `lifecycle.state` unchanged.
- **Consumer:** Institution `apps/agentos/util/SourceHealth.mjs` `mapFleetSessionHealth` accepts `running`, `stopped`, `failed` + `cleanupUnresolved`, and the card states. Anything else goes to `downgradeRuntime()`. `FleetStartPlan.partitionFleetStart` then excludes the seat because `runtime.state !== 'wired'`.
- **Brain itself would start the seat:** `start()` refuses only `ownershipUnresolved`, `stopping` and `cleanupUnresolved`, and `McpLaunchAdmissionService.reserve()` replaces a retained revoked generation.

**Design authority:** Brain already has a word for this fact. `adoptLeasedSeats` (`FleetLifecycleService.mjs:2058`) records a seat that died with no Fleet watching as `state: 'stopped'`, failureReason `'the seat exited while no Fleet server supervised it'`. The consumer's contract, the `SourceHealth.mapFleetSessionHealth` JSDoc, says: *"A matching stopped lifecycle remains a wired, honestly off session."* #1051's PR body, comments and reviews never mention `exited` (searched).

## The Fix

In `recoverRuntimeCustodyUnderGuard`, a gone, unadopted seat without unresolved cleanup becomes `state: 'stopped'`. Its failureReason is the `adoptLeasedSeats` wording when the saved state was `running`, and `null` otherwise, because such a seat had already ended under the previous Fleet. The two readers drop `exited`.

`isSeatRuntimeSnapshot` keeps accepting `exited` so that journals written since #1051 still open; rejecting it would freeze custody for every seat. The first boot of the fixed build re-persists those entries as `stopped`. The read value retires at the snapshot's next `version` bump.

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService.status(id).state`, which becomes cockpit `lifecycle.state` | `adoptLeasedSeats` precedent; `SourceHealth.mapFleetSessionHealth` contract | A recovered, gone seat reads `stopped` | A journal entry saying `exited` recovers as `stopped` | `recoverRuntimeCustody` JSDoc | AC-1–3 |

Decision Record impact: none. No ADR records the recovery state; this restores the word the lifecycle already used.

## Acceptance Criteria

- [ ] AC-1: A unit test drives a claude-desktop seat through Start → Stop → continuity handoff → a new issuer's `recoverRuntimeCustody` with the process gone. `status(id).state` is `stopped` and `failureReason` is `null`. The same test is red at `d6a905dc`.
- [ ] AC-2: A seat saved as `running` whose process is gone at recovery reads `stopped`, with the `adoptLeasedSeats` failureReason.
- [ ] AC-3: A journal whose seat entry says `exited` still opens and recovers to `stopped`, and the re-persisted snapshot says `stopped`.
- [ ] AC-4: No `state: 'exited'` write remains under `ai/`.
- ~~AC-5 (post-merge, installed): after an FM install carrying this, Ada's card reads off with ▶ enabled, and a Start succeeds.~~ Dropped: no open ticket owns FM release acceptance, so the PR cannot owe it. The installed result is reported here after the next FM install.

## Out of Scope

- Whether custody should retain terminal seats at all. Not rebuilding a record for a saved `stopped` seat would also unblock the journey, but it changes what #1051 retains (runtime pin, roster facts). That is #1051's design surface, not this defect.
- A shared lifecycle-state vocabulary in `src/fleet/contract/`, which would stop a producer from minting a word its consumer rejects without anyone noticing. Follow-up candidate.
- Institution: the mapper's fail-closed rejection is correct behavior.

## Avoided Traps

- **Teaching the roster `exited`:** adds a word on both sides for a fact Brain already names, and hides the next unknown word the same way.
- **Hand-editing the journal to drop the entry:** custody files carry issuer and epoch fencing; a bad edit freezes admission for every seat.

## Related

#1051 / #980 (durable custody, where `exited` came from) · #1136 / PR #1137 (the GPT-seat half of today's install; one FM install should carry both) · Institution `SourceHealth.mjs`.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-18T17:06:16Z; nearest #1136 and #1081, neither equivalent.
A2A sweep: last 30 messages at 17:06Z; Grace's #1136 / #1137 claims only.
MC sweep: "stopped seat cannot be started again from Fleet Manager after restart, lifecycle and runtime facts contradict", 6 results, no prior decision (nearest: a Codex Desktop helper-cleanup Stop-fleet class, 10-17).
Own-assignment sweep: 9 open, none overlapping.
Structure map: `ai/services/fleet` (existing files, no new file).

Retrieval Hint: "boot recovery stamps stopped seat exited roster contradiction Start disabled"

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-18T17:07:54Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T17:07:55Z @neo-opus-vega added the `bug` label
- 2026-10-18T17:07:56Z @neo-opus-vega added the `ai` label
- 2026-10-18T17:07:56Z @neo-opus-vega added the `regression` label
- 2026-10-18T17:07:56Z @neo-opus-vega added the `agent-os` label
- 2026-10-18T17:18:49Z @neo-opus-grace cross-referenced by #534
- 2026-10-18T17:19:18Z @neo-opus-vega cross-referenced by PR #1139
- 2026-10-18T17:34:43Z @tobiu referenced in commit `4c25b20` - "fix(fleet): a seat gone at the next boot recovers as stopped, not exited (#1138) (#1139)

Boot recovery (since #1051) stamped a saved seat whose process is gone 'exited', whatever state it was saved in. The FM roster's fail-closed mapper rejects that word, so a stopped seat's runtime read invalid and its Start was excluded; every boot re-persisted the record, so no restart cleared it.

Recovery now uses the word lease adoption already used for the same fact: 'stopped', with the unsupervised-exit reason only when the seat was saved running. The codec still reads 'exited' from journals the earlier builds wrote."
- 2026-10-18T17:34:43Z @tobiu closed this issue
- 2026-10-18T17:50:22Z @neo-opus-vega cross-referenced by PR #1141
- 2026-10-18T18:55:02Z @neo-opus-vega cross-referenced by #1142

