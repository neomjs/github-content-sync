---
id: 29
title: 'Fleet roster: 4-arrow grid navigation via the Gallery selection pattern'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-08-28T09:54:55Z'
updatedAt: '2026-08-28T14:38:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/29'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-28T14:38:23Z'
---
# Fleet roster: 4-arrow grid navigation via the Gallery selection pattern

## Context

Operator request (2026-08-28, post-split FM planning session): the fleet roster renders agent cards in a multi-column responsive grid, but keyboard navigation is the list default — arrow Up/Down only. With multiple columns on screen, Up/Down walks the flat item order and visually jumps across rows; Left/Right do nothing. The operator's explicit direction: "a navigation using all 4 arrow keys feels better. we already did this for the component.Gallery KeyNav / selection model => we probably don't need to re-invent the wheel."

## The Problem

A grid that renders two-dimensionally but navigates one-dimensionally breaks the spatial model the layout itself teaches. The cockpit is the operator's daily instrument; roster traversal (select an agent → detail rail, memories, mailbox mirror retarget) is its core loop, and today the keyboard path fights the visual layout instead of following it.

The engine already solved this class of problem once. Re-deriving 2D navigation ad hoc in the app layer would duplicate a shipped, tested pattern.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/List.mjs` — the animated roster list (`Neo.list.Component` family); cards flow into a responsive multi-column grid, so the column count is a function of container width, not a static config.
- `apps/agentos/view/fleet/roster/SelectionModel.mjs` — extends `Neo.selection.ListModel`, `singleSelect: true`, with the lifecycle-control carve-out (clicks inside a card's start/stop/restart cluster never touch selection). Its JSDoc states the current keyboard contract: "the Navigator addon moves item focus (the base list contract); Enter selects the focused row."
- Engine prior art: `src/selection/GalleryModel.mjs` — the shipped 2D selection model. `onKeyDownDown/Left/Right/Up` (lines ~95–123) route through `onNavKeyRow` / `onNavKeyColumn` keyed off the view's `orderByRow`, with a `stayInRow` config governing whether vertical moves stay in the column or wrap. Key bindings registered by the model itself (~line 218).
- Delta to respect: `Gallery` knows its geometry from config (`amountRows`); the roster's column count is emergent from rendered width. The 2D model here must derive columns-per-row from live geometry (rendered item boxes or the container's width breakpoint) before it can translate Up/Down into row moves.

## The Fix

Extend `AgentOS.view.fleet.roster.SelectionModel` (or a sibling it swaps in) with GalleryModel-pattern 4-arrow navigation adapted to the responsive grid:

1. Register Left/Right/Up/Down handlers following the `GalleryModel` shape (model-owned key bindings; no ad-hoc DOM listeners).
2. Derive the current column count from rendered geometry at navigation time (container width / item box), so reflow keeps navigation truthful at every breakpoint, including the 1-column narrow case (where Up/Down degrade to the flat order and Left/Right no-op or mirror them).
3. Preserve the existing contracts untouched: `singleSelect`, Enter-selects-focused, the lifecycle-control carve-out, and selection-driven pane retargeting.
4. Focus movement stays in the Navigator/focus layer; selection stays on Enter (matching the documented roster contract) — 4-arrow moves FOCUS, not selection.

## Acceptance Criteria

- [ ] All four arrow keys move item focus following the visual grid at ≥2 different column counts (wide + mid), verified with computed geometry receipts.
- [ ] 1-column narrow case: navigation degrades gracefully (no dead keys, no focus loss).
- [ ] Enter still selects the focused card; selection-driven panes retarget exactly as today.
- [ ] Lifecycle-control carve-out untouched (pointer path unaffected by the keyboard work).
- [ ] Unit coverage via the repo's custom Playwright unit config (`npm run test-unit`); no default `npx playwright test`.
- [ ] Both themes unaffected (no styling change in this ticket).

## Out of Scope

- Visual-contract repairs of the roster/activity surfaces (owned by #3).
- Multi-select semantics (no product meaning per the SelectionModel JSDoc).
- Focus-ring/appearance styling (focus VISUALS belong to #3 / the design-conformance epic #13).

## Avoided Traps

- Re-implementing 2D math in the app layer while `Neo.selection.GalleryModel` ships the pattern — the operator's explicit "don't re-invent the wheel" instruction.
- Porting `GalleryModel` verbatim: its geometry source (`amountRows` config) does not exist here; a static column config would lie at every resize.

## Related

- Parent: #10 (Epic: Fleet Manager cockpit UI/UX — Lane B).
- Sibling: #3 (visual contracts), #13 (design conformance epic).
- Engine prior art: `neomjs/neo` `src/selection/GalleryModel.mjs`, `src/component/Gallery.mjs`.

Live latest-open sweep: checked latest 30 open institution issues + the A2A herd window (60 min) at 2026-08-28T09:52Z; no equivalent found.

Origin Session ID: 55add047-b483-449f-b194-dce9a0df30d4

Retrieval Hint: "roster 4-arrow grid navigation GalleryModel responsive column derivation"


## Timeline

- 2026-08-28T09:54:56Z @neo-fable-clio added the `enhancement` label
- 2026-08-28T09:54:56Z @neo-fable-clio added the `agent-os` label
- 2026-08-28T09:54:57Z @neo-fable-clio added the `ai` label
- 2026-08-28T09:54:57Z @neo-fable-clio added the `design` label
- 2026-08-28T14:10:14Z @neo-fable-clio cross-referenced by PR #34
- 2026-08-28T14:26:43Z @tobiu referenced in commit `4dd68ef` - "fix(agentos): row navigation delegates to the list's Navigator envelope (#29)"
- 2026-08-28T14:38:23Z @tobiu referenced in commit `05a231d` - "Merge pull request #34 from neomjs/agent/29-roster-grid-keynav

feat(agentos): the roster grid gains its vertical axis — 4-arrow navigation (#29)"
- 2026-08-28T14:38:23Z @tobiu closed this issue
- 2026-09-04T18:29:58Z @neo-fable-clio cross-referenced by #64
- 2026-09-04T18:31:47Z @neo-fable-clio cross-referenced by PR #110

