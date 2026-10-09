---
id: 640
title: A parked Wake routes pane misses snapshots and the Reconnect re-drive
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-09T12:18:43Z'
updatedAt: '2026-10-09T14:08:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/640'
author: neo-opus-grace
commentsCount: 0
parentIssue: 505
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T14:08:27Z'
---
# A parked Wake routes pane misses snapshots and the Reconnect re-drive

## Context

Found while reviewing #634, which fixed this blind spot for Agent Detail (#632). Witnessed on 2026-10-09 with a throwaway unit spec on a real `FleetCockpit` at `95c4bff`, which is dev `32627ab` plus a spec-only commit; this accessor is unchanged on dev `0f04d98`. Clio routed the leaf here, under #505 beside #632.

## The Problem

| Step | Observed |
|---|---|
| W1: first reveal (`setItemAutoHidden` false) | The pane materializes: `neo-dock-pane-11`, its declared id |
| W2: `loadWakeRoutes()` | `pane.snapshot.count` = 1 |
| W3: park (`setItemAutoHidden` true) | `getWakeRoutesPane()` → `null`; `Neo.get(getPaneDeclaration('wakeRoutes').id)` → the same live instance |
| W4: `loadWakeRoutes()`, bridge answering count 2 | The owner's `wakeRoutesSnapshot.count` = 2; the pane stays at 1 |
| W5: reveal again | The same instance, still 1 |

What the operator sees: `LivenessController#reconnectFleet` re-drives Wake routes only through `cockpit.getWakeRoutesPane()?.onRefreshClick()`. Pressing Reconnect while Wake routes is parked skips the pane entirely. Its pre-reconnect envelope, such as a failed first read's `unavailable`, then survives the next reveal until a manual Refresh.

## The Architectural Reality

- `VesselContainer#getWakeRoutesPane` (`apps/agentos/view/fleet/cockpit/VesselContainer.mjs:512`) returns `this.vesselPane('wakeRoutes') || this.getReference('wakeRoutes')`. `getReference` answers only the projected tree.
- `Container#getPreservedItemIds` (`Container.mjs:791`) parks `detail`, `perspectives`, `defineAgent` and `wakeRoutes` when they leave the tree. `#seedPane` seeds fresh configs only ("a live instance, parked or projected, keeps its own state").
- `Controller#loadWakeRoutes` (`Controller.mjs:713`) is the only loader and is entered through the pane's `wakeRoutesRequest`. It pushes `snapshot` only into a pane it can resolve (`:740`).
- #634 resolved Detail through `Neo.get(this.getPaneDeclaration('detail')?.id)`. That is the engine's own rule: `Workspace#releaseDeclaredPanes` says "the component manager owns live instance lookup".
- Design authority: the accessor's own JSDoc ("so snapshot writes and the reconnect re-drive reach the pane in every phase"), and `reconnectFleet`'s ("a failed first read would otherwise pin its unavailable envelope forever"). The parked phase contradicts both.
- Perspectives and Add agent declare provider bindings. Memories and Catch up are not preserved; they rematerialize and seed. None of the four needs this change.

## The Fix

Change `getWakeRoutesPane()` to `this.vesselPane('wakeRoutes') || Neo.get(this.getPaneDeclaration('wakeRoutes')?.id) || null`, which is #634's line applied to Wake routes. Its JSDoc names the parked phase. There is no other source change.

## Acceptance Criteria

- [ ] AC-1: Unit test, red first on the current accessor. Reveal, park, then `loadWakeRoutes()` with a new snapshot. The parked pane holds the new snapshot, and the reveal shows the same instance holding it.
- [ ] AC-2: Unit test, red first. `reconnectFleet()` while Wake routes is parked reaches the parked pane's refresh, and its snapshot updates.
- [ ] AC-3: Unit test. Before the first materialization the accessor answers `null` and creates no pane.

## Out of Scope

- Memories, Catch up, Perspectives and Add agent (see above).
- The dock's auto-hide mechanics.

## Related

Parent #505 · #632 / #634 (the same fix for Detail)

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-09T12:17:13Z, plus `gh search issues "Wake routes"` (open). Only the broad rows (#505, #507, #424, #24, #12) matched; none covers this defect.
A2A sweep: the last 30 messages, all read states. No claim; Clio's routing asks for this leaf.
MC sweep: "Wake routes pane stale after hide and reveal, reconnect re-drive skipped, parked auto-hidden rail pane getReference null", 6 results. None is a prior decision on this surface.
Own-assignment sweep: #635, #633, #616, #490, #414, #11. None covers this surface.

Origin Session ID: e76b2469-377c-4fec-85a7-4c47b10269b9
Retrieval Hint: "Wake routes parked pane getReference null loadWakeRoutes reconnectFleet re-drive stale snapshot Neo.get declared id"


## Timeline

- 2026-10-09T12:18:45Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T12:18:45Z @neo-opus-grace added the `bug` label
- 2026-10-09T12:18:46Z @neo-opus-grace added the `agent-os` label
- 2026-10-09T12:18:46Z @neo-opus-grace added the `ai` label
- 2026-10-09T12:18:56Z @neo-opus-grace added parent issue #505
- 2026-10-09T12:31:28Z @neo-opus-grace cross-referenced by PR #641
- 2026-10-09T14:08:27Z @tobiu referenced in commit `436a580` - "fix(agentos): a parked Wake routes pane keeps its snapshot and the Reconnect re-drive (#640) (#641)

* fix(agentos): a parked Wake routes pane keeps its snapshot and the Reconnect re-drive (#640)

* test(agentos): the phase-blind accessor arm reads Wake routes' resident pane by its declared id (#640)"
- 2026-10-09T14:08:28Z @tobiu closed this issue

