---
id: 1552
title: A seat record with no process refuses the whole continuity handoff
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-09T23:04:48Z'
updatedAt: '2026-11-10T00:26:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1552'
author: neo-opus-ada
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-10T00:26:16Z'
---
# A seat record with no process refuses the whole continuity handoff

## Context

On 2026-11-09 at 21:05:47Z, the Fleet Manager quit for a package update (Brain `4ca0f506` → `67537f93`) and its `main.log` recorded:

`continuity handoff: refused at survivor-validation: seat neo-gpt-sophie is not a live original Start`

**Observed:**
- Sophie's seat was not running at the time.
- The successor recovered from `unclean-issuer-exit`, so all 7 running seats came back `stale`.
- Two of those seats then lost their memory-core MCP after a single 502 and could not reconnect until the operator stopped and started them: Vega at ~21:07Z and Grace at ~21:12Z (defect-notes `ff08823d`, `95cd1fd1`).

**Inferred, not read back:** Sophie's record was the `unavailable` record that `publishUnavailable` leaves when the Codex capability probe refuses a Start. The probe refused the installed build with `updater-disable-predicate-missing` until #1533, and the operator's Starts of the GPT seats failed that way. The old Fleet's process map ended with it, so the record itself cannot be read.

## The Problem

`handoffRuntimeCustody` skips only clean `stopped` records. Any other record without `runtimeStart` reaches `if (!start) throw new Error('seat … is not a live original Start')`. That check sits outside the per-seat catch #1166 added, so it refuses the handoff for every seat.

The method's JSDoc means that refusal for a **legacy seat**: a live seat with no generation-bound Start. A record whose Start was refused before spawn holds no process. It is not a seat at all, yet it is refused as one. So one refused Start turns the next FM update into an unclean exit for every running seat, the all-or-nothing outcome #1143 set out to remove.

## The Architectural Reality

- `ai/services/fleet/FleetLifecycleService.mjs` → `handoffRuntimeCustody` → `verifySeats`: the skip rule is `state === 'stopped' && !ownershipUnresolved && !cleanupUnresolved`, then comes `if (!start) throw`.
- Writers that leave a record with no process and no `runtimeStart`:
  - `publishUnavailable` (`state: 'unavailable'`). Its JSDoc says it runs before PAT resolution, Bridge-token minting and spawn. It has been in the file since `11552b08`, so it predates #1166.
  - `start`'s spawn-failure record (`state: 'failed'`, `pid: null`).
  - `adoptLeasedSeats`'s unreadable-lease record (`state: 'failed'`, `pid: null`, not adopted).
- A `failed` record can still claim a live process: `holdUnidentifiedSeat` (`ownershipUnresolved`) and owed Codex helper cleanup (`cleanupUnresolved`). The state label alone cannot tell the two apart.
- `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` pins the legacy rule: *a seat without a generation-bound Start still refuses the whole handoff, and the refusal revokes nothing*.

## The Fix

In `verifySeats`, pass over a record with no `runtimeStart` when it claims no process: no pid, no child handle, not adopted, no unresolved ownership or cleanup. Such a record holds no authority, so passing over it hands the successor nothing. A record without `runtimeStart` that does claim a process still refuses, so the legacy-seat rule is unchanged. The method JSDoc names the case.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService.handoffRuntimeCustody` survivor validation | #1143 (one seat's unprovable fact never refuses the update), #1166 | A record that holds no process is passed over; the handoff seals for the running seats | A live seat without a generation-bound Start still refuses | method JSDoc | AC-1, AC-2 |

**Decision Record impact:** aligned-with the ADR 0034 §2.5 amendment that #1143 cites (neomjs/neo#19857). No rule changes; a record stops being misread as a seat.

## Acceptance Criteria

- [ ] **AC-1:** One admitted running seat, plus the record `publishUnavailable` leaves for a refused Codex Start. The handoff seals (`handed-off`), the running seat stays `active` in the sealed snapshot, and the refused record carries nothing into it. Unit test, red on `dev`.
- [ ] **AC-2:** The pinned legacy refusal (a running seat whose `runtimeStart` is removed) still refuses the whole handoff, unchanged.
- [ ] **AC-3 (post-merge, needs an FM build):** the first FM update after a refused or failed Start seals the handoff, and `main.log` has no `refused at survivor-validation`. Owner: Ada.

## Out of Scope

- A `stale` seat whose dropped MCP cannot relaunch. That is #1143's design: a stale seat holds no grants. This ticket removes the unclean exit that made the seats stale.
- The FM pack recording the declared Electron pin rather than the one it packages (defect-note `6adc30b0`, Institution).
- Window-restore `-25211` refusals. They occur daily since at least 2026-11-02 and predate this install.

## Avoided Traps

- **Moving `!start` inside the per-seat catch.** That would carry a live legacy seat as unresolved instead of refusing it, which #1166 decided against and its test pins.
- **Skipping by state label (`unavailable` / `failed`).** A `failed` record can still claim a live process. The predicate reads the process claim, not the label.
- **Unreadable leases.** Passing over one hands nothing over; the successor reads the lease again itself. A bundle replacement under a live legacy process is still refused by the installer's census of processes that execute from the bundle.

## Related

#1143 (predecessor; its fix item 4), #1166, neomjs/neo-agent-institution#732 (epic), #1530 / #1533 (the Codex probe outage that left the record), #1531.

Sweeps:
- Live latest-20 open plus a closed search (`continuity handoff survivor`) at 2026-11-09T23:05Z; read #1143's Fix, Out of Scope and Contract Ledger. No equivalent found.
- Backlog: 170 open (red), so this is filed under the epic, neomjs/neo-agent-institution#732.
- A2A: no lane-claim or lane-intent on the handoff among the last 40 messages.
- Memory Core: no earlier decision on processless records.
- My open assignments: #1543 and #1531 are other surfaces.

Retrieval Hint: "continuity handoff refused at survivor-validation not a live original Start unavailable record"

Origin Session ID: b5820cc0-2d1e-4650-889d-1978d7adeb7e

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-09T23:04:48Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-09T23:04:50Z @neo-opus-ada added the `bug` label
- 2026-11-09T23:10:22Z @neo-opus-ada cross-referenced by PR #1553
- 2026-11-09T23:11:06Z @neo-opus-ada cross-referenced by #1531
- 2026-11-10T01:19:15Z @neo-opus-grace cross-referenced by #759

