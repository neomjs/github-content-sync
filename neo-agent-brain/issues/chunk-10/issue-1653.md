---
id: 1653
title: The open-work rotation skips a seat whose plan is exhausted until its reset
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-12T17:00:13Z'
updatedAt: '2026-11-12T20:28:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1653'
author: neo-opus-vega
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1646 Seats report their plan capacity to the fleet'
  - '[x] 807 Unowned and unreachable-holder PRs escalate to a rotation seat'
blocking: []
closedAt: '2026-11-12T20:28:57Z'
---
# The open-work rotation skips a seat whose plan is exhausted until its reset

## Context

#1646 gives every seat a plan-capacity reading in `who_is_online`'s verbose rows and a throttle verdict (`ai/services/fleet/capacityThrottleState.mjs#throttleStateFromCapacity`). The operator's friction of 2026-11-12 was open-work wakes reaching seats that are out of quota. This leaf is #1646's AC-4, moved here because it builds on #807's reachability map and must not touch owned-work routing.

- **Design authority:** n/a. New routing behavior; no existing behavior is called wrong.

## The Problem

A rotation wake (unowned work, outside-PR triage, run approval) can pick a seat whose weekly or 5-hour window is exhausted. The wake lands, the seat cannot act until its reset, and the work waits while another seat could take it.

## The Architectural Reality

- `ai/services/fleet/openWorkWakes.mjs#pickRotationSeat` skips a seat whose `routeOf(seat)` reads `unreachable`. The route map comes from `readUndeliverable` and is cached for 5 minutes (`wireFleetOpenWorkWakes.mjs`).
- `routeOf` also gates reviewer and author wakes and pending merges. Adding an exhausted seat to the undeliverable map would therefore defer the work it holds as well, which this leaf must not do: owned work keeps its holder.
- #807 (PR #1652) rebuilds `readUndeliverable` from `who_is_online`'s verbose rows; after #1646 those rows carry `capacity`.

## The Fix

The rotation picker gets its own reader of exhausted seats: a seat whose verdict is `rate-limited` and whose binding window names a `resetsAt` still ahead, skipped by `pickRotationSeat` only, until that instant. Reviewer-, author- and merge-held wakes keep routing through `routeOf` unchanged.

## Acceptance Criteria

- [ ] The open-work rotation skips a `rate-limited` seat until its `resetsAt` and picks it again afterwards (unit arms on the picker with a clock).
- [ ] Reviewer-, author- and merge-held wakes for an exhausted seat are unchanged, and participation is never written.
- [ ] A reading without `resetsAt`, or a stale or unobserved one, never starts a skip.
- [ ] Post-merge: after a plane cut, one rotation wake on the plane that skipped an exhausted seat. The same cut takes #1646's receipt: real seats' capacity states in `who_is_online` and their throttle verdicts on the Fleet roster.

## Intake (2026-11-12, author, earlier-session ticket)

- **Drift probe fired:** since `createdAt`, `openWorkWakes.mjs` and `wireFleetOpenWorkWakes.mjs` moved, in #1652 only (`dc3082cc`, the blocker this ticket named). The full gate ran. Verdict: valid as written, sharpened below.
- **Readiness:** the reader needs #1646's `capacity` rows and `throttleStateFromCapacity`, both still on PR #1654's branch. The missing `blocked_by #1646` edge is now set, so the build starts when #1646 closes.
- **Prescription checked:** `ai/services/fleet/openWorkWakes.mjs#pickRotationSeat` owns the skip. Its only caller is `planOpenWorkWakes`'s `rotate()`, which serves both rotation actions and `holder-unreachable` escalations, while holder wakes call `routeOf` directly. Shape:
  - `capacityThrottleState.mjs` gains `exhaustedUntil(capacity, now)`, the latest `resetsAt` of a fresh binding window at 100 %, and a presence-level map beside it with pool windows applied. It shares one binding-window filter with `throttleStateFromCapacity` rather than re-deriving which windows bind a seat's model.
  - `planOpenWorkWakes` takes the seat-to-instant map and gives `rotate()` a route that reads `unreachable` while `exhaustedUntil > now`. The roster and its hash offset stay whole, so a skipped seat passes its pick to its successor, as an unreachable one does.
  - `createOpenWorkWakeRound` reads the map with the route map, under the same TTL, from the same `readPresence` answer.
- **Not touched:** participation, bench, and every `routeOf` call outside `rotate()`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `capacityThrottleState.mjs` `exhaustedUntil(capacity, now)` / `exhaustedUntilOf(presence, now)` | #1646's reading: a `who_is_online` row's `capacity`, with `pools` | The latest `resetsAt` among a seat's fresh binding windows at 100 %, pool windows applied, keyed by presence identity | A stale, unobserved or reset-less reading gives no entry, so no wait | JSDoc on both | `capacityThrottleState.spec.mjs` (#1653 arms) |
| `openWorkWakes.mjs` `planOpenWorkWakes({exhaustedUntil})` | this ticket | Rotation only: `rotate()` reads a seat as unreachable while `exhaustedUntil[seat] > now`; the roster and its offset stay whole; holder wakes and pending merges keep `routeOf` | An absent or `null` map skips nobody | `@param` JSDoc | `openWorkWakes.spec.mjs` (#1653 arm) |
| `wireFleetOpenWorkWakes.mjs` `createOpenWorkWakeRound({readExhausted})` and the wired presence refresh | this ticket | One presence answer per route refresh serves the route map and the reset map, cached with the routes TTL | A failed read or a malformed map reads as unknown (`null`): no skip | `@param` JSDoc | `wireFleetOpenWorkWakes.spec.mjs` (#1653 arm, one read) |

## Out of Scope

- Re-seating owned work whose holder is out of quota past its SLA: the sibling #1646 names.
- Bench and participation (#28).

## Related

#1646 (the reading and the verdict), #807 / PR #1652 (the verbose-row map, blocks this), #1642.

Live latest-open + closed sweep: latest 20 of 158 open issues (red band; #1646 is the owning ticket this leaf splits from, #1642 and #1651 are adjacent and different) + closed searches ("rotation rate-limited", "skip exhausted seat", "quota reset rotation") + A2A claims of the last hour + own assignments at 2026-11-12T16:58Z; no equivalent or prior exclusion found.

Origin Session ID: 2051845f-04a4-4248-a5b5-a633fd710e5a
Retrieval Hint: "open-work rotation skip rate-limited seat until resetsAt capacity"


## Timeline

- 2026-11-12T17:00:13Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-12T17:00:15Z @neo-opus-vega added the `enhancement` label
- 2026-11-12T17:00:15Z @neo-opus-vega added the `ai` label
- 2026-11-12T17:00:16Z @neo-opus-vega added the `agent-os` label
- 2026-11-12T17:00:21Z @neo-opus-vega marked this issue as being blocked by #807
- 2026-11-12T17:01:00Z @neo-opus-vega cross-referenced by #1646
- 2026-11-12T17:02:51Z @neo-opus-vega cross-referenced by PR #1654
- 2026-11-12T19:25:59Z @neo-opus-vega marked this issue as being blocked by #1646
- 2026-11-12T20:00:03Z @neo-opus-vega cross-referenced by PR #1657
### @neo-opus-vega - 2026-11-12T20:10:32Z

**Handover (session sunset, solo-refresh, 2026-11-12 ~20:1xZ), owner Vega.**

- **State:** PR #1657 at `4bd24e94` (rebased onto `a16eb4e4`). CI green 20:07Z, CLEAN, Euclid requested as primary reviewer (cross-family; he holds the #1654 context).
- **Shipped shape:** the "Intake" section above. `capacityThrottleState.mjs` has one private `bindingWindows` filter feeding `throttleStateFromCapacity`, `exhaustedUntil` and `exhaustedUntilOf`. `planOpenWorkWakes`'s `rotate()` gets the rotation-only route. The round reads the reset map with the route map, from one presence answer per refresh.
- **Evidence:** fleet unit folder 2,251 passed, 3 skipped at the head. Two red-first controls bite: the rotation route reverted, and one presence read per reader.
- **Pickup:** answer Euclid's review (`review-response-protocol.md`); after merge, take the post-merge receipt. The plane already runs `a16eb4e4`, but every seat's `capacity` still reads `unobserved` (partial receipt on #1646, comment 6834685990). Both receipts wait for a seat Start whose settings carry the capacity hook, the same trigger as #1615's census.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-12T20:28:57Z @tobiu referenced in commit `857fe41` - "feat(fleet): the rotation skips a seat whose plan is exhausted until its reset (#1653) (#1657)

The open-work rotation passed wakes to a seat whose weekly or 5-hour window was full, so the work waited for
that seat's reset while another seat could take it. The rotation now reads each seat's latest full binding
window reset from the same verbose presence answer as the route map, and passes such a seat over until that
instant, as it passes an unreachable one. The roster and its hash offset stay whole. Work a seat holds as
reviewer or author still routes through routeOf alone, and participation is never written.

- capacityThrottleState: bindingWindows is the one filter of the windows that bind a seat now;
  throttleStateFromCapacity, exhaustedUntil and exhaustedUntilOf read through it. A full window
  without a resetsAt, or a stale or unobserved reading, starts no wait.
- openWorkWakes: planOpenWorkWakes takes exhaustedUntil and gives rotate() a rotation-only route.
- wireFleetOpenWorkWakes: the round reads the reset map with the route map, under the same TTL;
  the wired reader serves both maps from one presence answer."
- 2026-11-12T20:28:59Z @tobiu closed this issue

