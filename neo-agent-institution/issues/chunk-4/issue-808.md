---
id: 808
title: The cockpit's detail reveal reads its end cause and pops out in place
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-25T02:44:23Z'
updatedAt: '2026-10-25T16:08:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/808'
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
closedAt: '2026-10-25T16:08:51Z'
---
# The cockpit's detail reveal reads its end cause and pops out in place

### Context
neo #20103 (merged 2026-10-25T02:36Z as `2c069de412`, resolving neo #20095) gives the auto-hide reveal two things the cockpit currently works around:
- **A pop-out on the reveal toolbar** (`.neo-dashboard-dock-reveal-pop-out`). It is gated as the docked header gates it and runs the docked pop-out's own transaction.
- **A `cause` on every `dockRailRevealChange` that ends a reveal.** The cause names who ended it:
  - the person: `escape`, `tab`, `outside-press`, `focus-leave`, `pointer-leave`, `hover`;
  - the application: `release`, `hold`;
  - the item leaving its rail: `item-left`.

  It is `null` when no reveal ends.

Both are the engine half of the #791 design (Clio's read, Q1 (a)).

### The Problem
- **A test helper that pins before every pop-out.** `test/playwright/fixtures.mjs#pinRevealedDetail` docks the revealed detail through the reveal's pin before six arms pop it out, because "the reveal toolbar carries no pop-out". Its doc says: "Delete with its callers once the reveal toolbar carries the item's header actions." The callers are `FleetCockpitDrillRoundTripNL`, `FleetCockpitKineticNL`, `FleetCockpitNWindowNL`, `FleetCockpitPopOutNL`, `FleetPermanenceMatrixRow4NL` and `FleetPerspectiveCaptureVesseledNL`.
- **A flag that stands in for the cause.** `Container#dismissingDetailReveal` is held around `dismissDetailReveal()` so that `CockpitPerspectives.detailRevealChange` can tell the cockpit's own release from a person's dismissal. `cause === 'release'` says that directly.
- **A latent break.**
  - `detailRevealChange` treats every end of the detail's reveal as the person's and clears the selection, unless the inspector is no longer railed.
  - The engine's reveal pop-out ends the reveal with `item-left` before its detach commits: in `HeaderActions#popOutRailItem`, `handOff` runs `itemCleared` ahead of `onDockTearOutTerminal`. At that moment `railsInspector` still holds.
  - So, reading the source, the first engine pin past `2c069de412` makes a pop-out from the reveal clear the selection. The PR proves this red first.

### The Architectural Reality
- `apps/agentos/util/CockpitPerspectives.mjs#detailRevealChange` receives `{next, previous, rail, railItem}` and now also `cause`.
- `apps/agentos/view/fleet/cockpit/Container.mjs` holds `dismissDetailReveal()`, the `dismissingDetailReveal` member and `listeners: {dockRailRevealChange: 'onDetailRevealChange'}`.
- **Engine pin.** Since #807 merged (`84e86b5`), the lock resolves `neo.mjs` to `0c1a373918`. This change needs a dev commit that contains `2c069de412`. The lock moves to `a97cb2d5f8`. Its `src/` delta from `0c1a373918` is the eight dock files of #20103 plus `calendar/view/week/Component.mjs` (#19958), with no `package.json`, SCSS or version change.
- **A known engine edge.** A detach refused after admission still ends the reveal with `item-left` while the item stays on its rail (Maryam's review of neo #20103, 5640968779). The cockpit keeps the selection, and a press on the selected card reveals the detail again (`detailPress`).

### The Fix
1. **The lock.** `neo.mjs` moves to a dev commit containing `2c069de412`, through the pin checklist, with the visual and Neural Link suites compared at both engines.
2. **The handler.** `detailRevealChange` reads `cause`: a person cause ends the selection, and `release` and `item-left` keep it. `dismissingDetailReveal` goes, and `Container#dismissDetailReveal` with it: the cockpit ends the reveal with `dismissRailReveal('detail')`, which the rail reports as `release`.
3. **The helper.** `pinRevealedDetail` is deleted, and each of the six arms pops the detail out from the reveal's toolbar.
4. **A red in a touched file.** `FleetCockpitDrillRoundTripNL` already fails on dev at both engines (`agent-detail-dark-narrow-271` differs by 4354 px; defect note of 2026-10-25). Its screenshot is re-captured after a design read of the render.
5. **The return** (found by the arms). A reveal pop-out keeps the item `autoHidden`, so closing the vessel brings the inspector back to its rail. While a resident is selected, the cockpit reveals it again (`onDockPaneReturn`).
6. **A vesseled inspector is not revealed from its rail** (found by the arms). A perspective restored while the inspector is away puts it back in its rail before the vessel returns the pane. A reveal at that moment shows a placeholder. On the return, the rail sees the same item id, keeps the placeholder, and the live pane stays in the closed vessel's viewport.

### Acceptance Criteria
- [ ] The lock resolves `neo.mjs` to a dev commit that contains `2c069de412`.
- [ ] Popping the detail out from the reveal's toolbar keeps the selection. This NL arm is red with the old handler at the new engine and green with the fix.
- [ ] Escape and the rail tab still end the selection, and an outside press and the cockpit's own release still keep it (`FleetCockpitDrillNL`). *(Corrected 10-25: as filed, this said an outside press ends the selection. #791's design and its arms keep it.)*
- [ ] Closing the vessel of a reveal pop-out brings the same inspector back to its rail, revealed while a resident is selected (`FleetCockpitPopOutNL`).
- [ ] A perspective restored while the inspector is away does not reveal it, and after the return the same instance mounts in the cockpit window (`FleetPerspectiveCaptureVesseledNL`).
- [ ] `pinRevealedDetail` and `dismissingDetailReveal` are gone, and the six arms pop out from the reveal and pass.
- [ ] `FleetCockpitDrillRoundTripNL` passes, with its two 271 px screenshots (dark and light) re-captured after a design read.
- [ ] The visual suite and the NL battery give the same per-test results at both engines, except where this change means them to differ.

### Out of Scope
- The other stale goldens and NL failures on dev (the defect notes of 2026-10-25). The one in a touched file is in scope.
- The engine's order on a refused detach: the reveal ends before the commit is known.
- Merging before walk candidate 2 is cut. This PR merges after the cut, so Tuesday's walk runs on one stable candidate.

### Related
- neo #20095 and neo #20103 (the engine half); #791 and #805 (the cockpit's reveal); #806 and #807 (the pin before this one).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues and the open PRs at 2026-10-25T02:43Z, and found no equivalent. #795, a red in one of the six callers, no longer reproduces on dev: it passed 2 of 2 runs at `f8bfa66` with both engines, and `84e86b5` is that tree with the new lock. The A2A claim sweep (latest lane-claims) found no competing claim.

Origin Session ID: bb7f4705-9365-478d-879f-5276a3839427
Retrieval Hint: "cockpit detail reveal end cause item-left selection pinRevealedDetail"



## Timeline

- 2026-10-25T02:44:24Z @neo-opus-ada added the `enhancement` label
- 2026-10-25T02:44:25Z @neo-opus-ada added the `ai` label
- 2026-10-25T02:44:25Z @neo-opus-ada added the `testing` label
- 2026-10-25T02:44:36Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-25T02:45:02Z @neo-opus-ada cross-referenced by #795
- 2026-10-25T15:15:29Z @neo-opus-ada cross-referenced by PR #810
- 2026-10-25T15:31:34Z @neo-opus-ada referenced in commit `bc25686` - "Merge dev into ada/808-reveal-end-cause: #809's operator-inbox publish card and its goldens (#808)"
- 2026-10-25T15:31:34Z @neo-opus-ada referenced in commit `29799e3` - "test(agentos): re-capture five drifted cockpit goldens after design reads, and re-stamp (#808)

The reveal handler and the cockpit container are style-owning inputs, so the stamp moves with them. Before
re-stamping, each failing golden got a design read:
- the merge-queue heads (open, Stage-enabled, its light twin) and the two setup rows differ only in the
  list focus ring, which the engine changed;
- the Memories pane at 720 (and its light twin) carries the rewritten head copy.

pane-agent-detail is left red: at 1280 the capture loses the pane's inset and clips the avatar at the top.
That is not drift to accept."
- 2026-10-25T15:55:25Z @neo-opus-ada referenced in commit `e9b731b` - "test(agentos): pane-agent-detail frames the reveal's pane slot, which carries the inspector's inset (#808)

Since the selection reveals the railed inspector (#805), its 12 px inset belongs to the reveal's pane slot,
and `.fm-agent-detail` itself has no padding. The element capture cropped at the detail's own box, so it
looked inset-less, with the avatar at the edge. The page at 1280 shows the full inset and the whole avatar.
The capture now frames the slot, as the operator reads it (306×671), and is re-captured after a design
read."
- 2026-10-25T16:08:51Z @tobiu closed this issue
- 2026-10-25T16:08:51Z @tobiu referenced in commit `7ddcd78` - "feat(agentos): the detail reveal reads its end cause, pops out in place and comes home revealed (#808) (#810)

* build(deps): pin the engine to neo dev a97cb2d5f8, which carries the reveal pop-out and its end causes (#808)

The lock resolved neo.mjs to 0c1a373918, which predates neo #20103. The src/ delta to a97cb2d5f8 is the
eight dock files of #20103 and calendar/view/week/Component.mjs (#19958): no package.json, SCSS or
version change.

* feat(agentos): the detail reveal reads its end cause, pops out in place and comes home revealed (#808)

CockpitPerspectives#detailRevealChange ends the selection only on the person's causes (personEnds). The
cockpit's own end (`release`) and the item leaving its rail (`item-left`, a pop-out from the reveal's
toolbar) keep it, so Container#dismissingDetailReveal and #dismissDetailReveal go: the cockpit ends
the reveal with dismissRailReveal('detail').

Two behaviors the reveal pop-out needed, found by the arms:
- Container#onDockPaneReturn reveals the inspector again when it comes home to its rail while a
  resident is selected.
- CockpitPerspectives#revealsFromRail keeps a vesseled inspector from being revealed. A perspective
  restored while it is away rails it first, and a reveal then cached a placeholder that kept the live
  pane in the closed vessel's viewport after the return.

The six arms that pinned the reveal before popping out now pop out from it: three by the toolbar's
button, three by Workspace#popOutRailItem. pinRevealedDetail goes. FleetCockpitPopOutNL asserts that
the pop-out keeps the selection: red with dev's handler at this engine, green here. The two 271 px
goldens of FleetCockpitDrillRoundTripNL are re-captured. Their drift is the portrait, the rename
action (#760) and the thought-stream copy, all merged earlier.

* test(agentos): re-capture five drifted cockpit goldens after design reads, and re-stamp (#808)

The reveal handler and the cockpit container are style-owning inputs, so the stamp moves with them. Before
re-stamping, each failing golden got a design read:
- the merge-queue heads (open, Stage-enabled, its light twin) and the two setup rows differ only in the
  list focus ring, which the engine changed;
- the Memories pane at 720 (and its light twin) carries the rewritten head copy.

pane-agent-detail is left red: at 1280 the capture loses the pane's inset and clips the avatar at the top.
That is not drift to accept.

* test(agentos): pane-agent-detail frames the reveal's pane slot, which carries the inspector's inset (#808)

Since the selection reveals the railed inspector (#805), its 12 px inset belongs to the reveal's pane slot,
and `.fm-agent-detail` itself has no padding. The element capture cropped at the detail's own box, so it
looked inset-less, with the avatar at the edge. The page at 1280 shows the full inset and the whole avatar.
The capture now frames the slot, as the operator reads it (306×671), and is re-captured after a design
read."

