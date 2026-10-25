---
id: 1260
title: The roster offers Start while a booting Fleet refuses every Start
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-24T22:41:41Z'
updatedAt: '2026-10-25T00:44:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1260'
author: neo-opus-ada
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
closedAt: '2026-10-25T00:44:59Z'
---
# The roster offers Start while a booting Fleet refuses every Start

## Context

Since b08d81a3 (#1224, for #1198) a packaged Fleet's wire answers before its runtime verify and recovery run, and every Start is refused until recovery settles: `Fleet launch continuity is not ready.` from `McpLaunchAdmissionService#reserve`. Measured windows (verify + recovery):

- the installed FM's boot of 2026-10-20 16:12Z: 12,225 ms + 9,071 ms (`main.log` phase lines);
- the packaged rehearsal, 2026-10-24: 9,500 ms at boot, 16,631 ms after its unclean restart.

During the window the roster stamps `launchRefusal: null` on a stopped seat, so its card offers Start and the press is refused. The installed FM's log holds no refused Start since Oct 20, so no person has pressed Start inside the window yet.

The pre-install rehearsal (neomjs/neo-agent-institution#736, `npm run rehearse:packaged`) has been red on every candidate since #1224. The seat journey's first Start arrives about 0.4 s after the wire answers. Measured twice on a candidate of Institution 66bfd19, Brain 97d3c695 and Engine 13.3.1: both seats fail with `start: the wire call failed: fleet: 'startAgent' failed`. The last green candidate ran Brain 2b0e2ccd, which predates #1224 (2026-10-19). A probe build of Brain 9a8cb248 (it contains #1224) whose journey retried every failed Start (never committed) passed every step, admitted after 9,733 ms, so the window is the only red that run saw.

## The Problem

A verb the cockpit offers is refused for a reason that ends on its own seconds later. The window grows with the runtime: 23,747 files and 965 MB per #1198. Before Tuesday's walks it blocks the pre-install gate for every FM candidate.

## The Architectural Reality

- `FleetControlBridge#fleetRoster` stamps `launchRefusal: launchRefusalOf(agent, participation)`. Its JSDoc defines the field as "why the start verb refuses the seat, or `null`".
- `launchRefusalOf` (`src/fleet/contract/launchAuthority.mjs`) is per-seat by design: "a predicate over a registry row and the seat's participation, and nothing else". The spawn path asks it between admission and spawn.
- The start verb also refuses at the boot gate:
  - `McpLaunchAdmissionService#reserve`, through `startAgentProvisioned`;
  - `FleetLifecycleService#start`, which refuses while `continuityPhase !== 'ready'`.
- The phases:
  - `pending-recovery`, from `openRuntimeCustody({awaitRecovery: true})`;
  - `recovering`;
  - `ready`;
  - `unavailable`, after `failRuntimeVerification` or a failed recovery.
- `devFleetServer.mjs`, step 6: the transport listens, then `runtime.verify()`, then `recoverRuntimeCustody`.
- The Institution card's toggle (`apps/agentos/view/fleet/roster/card/Container.mjs`) carries "the Fleet's own words for why its Start refuses this seat: Start stays closed before the click". So the cockpit needs no change.
- `seatJourney.mjs` (`start`) calls `fleet.startAgent(id)` right after define, and reads rows from `fleetRuntimeStatus`.
- Design authority: #1198 AC-4, "the transport listens before any payload inventory runs, and a Start is refused until recovery settles." The refusal stays. This ticket only makes `fleetRoster` report it before the click, as its JSDoc already promises.

Decision Record impact: aligned-with ADR 0034 §2.5 (authority returns only after a byte check) and #1198 AC-4.

## The Fix

1. **`fleetRoster` reports the gate in `launchRefusal` while it is closed.**
   - A seat's own refusal comes first, since it outlives the gate.
   - Otherwise the gate's sentence: while `pending-recovery` or `recovering`, that the Fleet is verifying its runtime and Start opens when it finishes; while `unavailable`, the named refusal.
   - The gate is read from the lifecycle beside `getLaunchAdmission()`. `launchRefusalOf` stays per-seat.
2. **`runSeatJourney` waits the cockpit's way.** Before each Start it waits until the seat's roster row carries no `launchRefusal`, bounded by its `timeoutMs`. A refusal that stays becomes that step's failure reason.
3. **The journey's CI arm (#1164) gains a Fleet whose gate opens late.**

## Acceptance Criteria

- [ ] AC-1 (unit): `fleetRoster` rows carry the gate's sentence in `launchRefusal` while `continuityPhase` is `pending-recovery` or `recovering`, and the named refusal while it is `unavailable`. They carry `null` once it is `ready`. A seat's own refusal is unchanged and wins.
- [ ] AC-2 (unit, red on base): against a Fleet whose Start gate opens after a delay, the seat journey sends no Start while the gate is shut, and every step it reaches passes. A refusal that stays fails the step in the roster's words, and no Start goes out. On base the first Start fails. (Every step through recovery is AC-3's.)
- [ ] AC-3 (L3, packaged): `npm run rehearse:packaged` on a candidate built from this head returns `ok: true`. The base candidate above returns the two `start` failures.
- [ ] AC-4 [post-merge]: on the installed FM's next boot, a stopped seat's Start stays closed with the gate's sentence until the `recovery` phase line, then opens.

## Out of Scope

- **Holding a Start until the gate opens.** It would change #1198 AC-4's refusal.
- **The Institution rehearsal's `readStartOffers`**, which reads the roster state alone. Honouring `launchRefusal` there, with the same wait, is a follow-up in neomjs/neo-agent-institution once this lands.

## Avoided Traps

- **Retrying every failed Start in the journey**, as the probe did: it hides every other Start failure.
- **Putting the gate into `launchRefusalOf`**: the predicate is per-seat, and the spawn path asks it.

## Related

#1198 / #1224 (the gate) · #1164 / #1165 (the journey) · neomjs/neo-agent-institution#736 (the rehearsal) · ADR 0034 §2.5

Live latest-open sweep: checked latest 20 open issues at 2026-10-24T22:39Z (newest #1258); no equivalent found.
A2A claim sweep: last 30 messages, all read states; no claim on the boot Start gate or the seat journey.
MC sweep: "Start refused right after Fleet Manager boot, launch continuity is not ready, cockpit offers Start while Fleet verifies runtime", 10 results, no prior decision found.
Own-assignment sweep: 22 open, none overlapping (nearest #571, #1230).
Structure map: N/A, no new file. Owners are `ai/services/fleet/FleetControlBridge.mjs` (the roster assembler) and `ai/services/fleet/seatJourney.mjs`.

Origin Session ID: 21d750a5-78cd-44dc-9f8f-1cbfcc2ca1d6

Handoff Retrieval Hints: `query_raw_memories("packaged rehearsal Start refused launch continuity not ready boot window roster launchRefusal")`



## Timeline

- 2026-10-24T22:41:42Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-24T22:41:42Z @neo-opus-ada added the `bug` label
- 2026-10-24T22:41:42Z @neo-opus-ada added the `ai` label
- 2026-10-24T22:41:42Z @neo-opus-ada added the `regression` label
- 2026-10-24T22:41:42Z @neo-opus-ada added the `agent-os` label
- 2026-10-25T00:17:02Z @neo-opus-ada cross-referenced by PR #1262
- 2026-10-25T00:44:59Z @tobiu closed this issue
- 2026-10-25T00:44:59Z @tobiu referenced in commit `a60f9ea` - "fix(fleet): the roster names the Fleet's Start gate, and the seat journey waits for it (#1260) (#1262)"

