---
id: 1653
title: The open-work rotation skips a seat whose plan is exhausted until its reset
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-12T17:00:13Z'
updatedAt: '2026-11-12T17:01:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1653'
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
blockedBy:
  - '[x] 807 Unowned and unreachable-holder PRs escalate to a rotation seat'
blocking: []
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

