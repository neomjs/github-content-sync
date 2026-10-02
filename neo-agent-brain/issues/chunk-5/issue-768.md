---
id: 768
title: The Fleet stops arming claude-desktop on osascript once a Fleet-launched Claude seat arms itself at SessionStart
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T16:52:33Z'
updatedAt: '2026-10-02T16:52:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/768'
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
  - '[x] 766 Seat hooks resolve no plane: they read Fleet-transport leaves'
blocking: []
---
# The Fleet stops arming claude-desktop on osascript once a Fleet-launched Claude seat arms itself at SessionStart

## Context

Brain #562 / PR #752 (merged 2026-10-02) moved Claude seats to a pull route: the projected `wakeArmingHook` subscribes `SENT_TO_ME` with `harnessTarget: 'none'` and retires the seat's own `osascript` `SENT_TO_ME` routes, and the `asyncRewake` listener polls `poll-digest`. #705's `armFleetSeatWake` still subscribes `claude-desktop` on `osascript` at a Fleet Start (`GUI_WAKE_DISPATCH`), so a Start re-creates the route the seat's next SessionStart retires, and the Fleet's status reads `ready` for a route that is gone (Ada's Tier 2.5 of 2026-10-02 13:04Z; decided 16:23Z: keep the route now, drop it as a leaf, never arm the pull route from the Fleet).

## The Problem

Today the drop would strand a Fleet-launched Claude seat: its hooks read `fleet.planeBase` / `fleet.planeBearer`, which the Fleet's launch does not inject, so `wakeArmingHook` reported UNARMED in every one of 41 measured transcripts — the seat cannot yet retire the osascript route itself (#766). Until it can, the osascript route is the only route such a seat has; the stale `ready` is bounded to Start → next SessionStart.

## The Fix

With #766's PMV-1 proven (a Fleet-launched Claude seat arms at SessionStart): drop `claude-desktop` from `GUI_WAKE_DISPATCH` so `armFleetSeatWake` answers `null` for it, the way OpenCode keeps its own route; the Fleet's wake status for a Claude seat then reads the seat's own arming receipt on the plane, never a route the Fleet armed.

## Acceptance Criteria

- [ ] `armFleetSeatWake` answers `null` for a `claude-desktop` seat; the `armFleetSeatWake` arms cover it beside OpenCode (red-first: today it subscribes).
- [ ] A Fleet Start of a Claude seat leaves no `osascript` `SENT_TO_ME` subscription for that seat on the plane (unit over the subscription writes; installed receipt in PMV).
- [ ] The Fleet's wake status for a Claude seat reads the seat's own arming receipt (`wakeArmingHook` ARMED), not a Fleet-armed route.

## Out of Scope

- #766 itself (the seat-side plane leaves and the `start` injection beside `resolvedMcpCredential`).
- Codex and OpenCode routes.

## Related

#562 / PR #752 · #705 · #728 · #766 (blocks this)

Owner: Vega (self-assigned on creation); blocked by #766 (native dependency).

Live latest-open sweep: the open Brain queue at 16:52Z carries #766 (the prerequisite) and no leaf on the Fleet's claude-desktop arming; `gh issue list --search "claude-desktop GUI_WAKE_DISPATCH OR armFleetSeatWake OR osascript route"` returns nothing besides it.

Origin Session ID: 60d9be31-4233-40fd-9b07-6ee6a9ebf6cf
Retrieval Hint: "armFleetSeatWake claude-desktop osascript GUI_WAKE_DISPATCH drop pull route SessionStart #766"


## Timeline

- 2026-10-02T16:52:34Z @neo-opus-vega added the `enhancement` label
- 2026-10-02T16:52:34Z @neo-opus-vega added the `ai` label
- 2026-10-02T16:52:35Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T16:52:38Z @neo-opus-vega marked this issue as being blocked by #766
- 2026-10-02T17:45:12Z @neo-gpt-emmy cross-referenced by PR #771
- 2026-10-02T17:49:46Z @neo-opus-ada cross-referenced by #30

