---
id: 808
title: The cockpit's detail reveal reads its end cause and pops out in place
state: OPEN
labels:
  - enhancement
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-25T02:44:23Z'
updatedAt: '2026-10-25T02:44:36Z'
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
- **Engine pin.** Since #807 merged (`84e86b5`), the lock resolves `neo.mjs` to `0c1a373918`. This change needs a dev commit that contains `2c069de412`.
- **A known engine edge.** A detach refused after admission still ends the reveal with `item-left` while the item stays on its rail (Maryam's review of neo #20103, 5640968779). The cockpit keeps the selection, and a press on the selected card reveals the detail again (`detailPress`).

### The Fix
1. **The lock.** `neo.mjs` moves to a dev commit containing `2c069de412`, through the pin checklist, with the visual and Neural Link suites compared at both engines.
2. **The handler.** `detailRevealChange` reads `cause`: a person cause ends the selection, and `release` and `item-left` keep it. `dismissingDetailReveal` goes.
3. **The helper.** `pinRevealedDetail` is deleted, and each of the six arms pops the detail out from the reveal's toolbar.
4. **A red in a touched file.** `FleetCockpitDrillRoundTripNL` already fails on dev at both engines (`agent-detail-dark-narrow-271` differs by 4354 px; defect note of 2026-10-25). Its screenshot is re-captured after a design read of the render.

### Acceptance Criteria
- [ ] The lock resolves `neo.mjs` to a dev commit that contains `2c069de412`.
- [ ] Popping the detail out from the reveal's toolbar keeps the selection. This NL arm is red with the old handler at the new engine and green with the fix.
- [ ] Escape, the rail tab and an outside press still end the selection, and the cockpit's own release still keeps it (NL arms).
- [ ] `pinRevealedDetail` and `dismissingDetailReveal` are gone, and the six arms pop out from the reveal and pass.
- [ ] `FleetCockpitDrillRoundTripNL` passes, with its screenshot re-captured after a design read.
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

