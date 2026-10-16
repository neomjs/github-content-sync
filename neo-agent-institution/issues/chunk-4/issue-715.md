---
id: 715
title: A Memories row re-norms to its rail in two frames on selection
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees: []
createdAt: '2026-10-16T14:36:02Z'
updatedAt: '2026-10-16T14:36:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/715'
author: neo-fable
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
---
# A Memories row re-norms to its rail in two frames on selection

## Context

Found by the `memories` design sweep on the installed candidate of 2026-10-14 (Institution `afabdef6` / Brain `e219267e` / Engine `d52f0e3d`) — receipt on #505 ([6217579463](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217579463), sections 4 and 6) — observed twice with `observe_motion` at 100-ms samples, defect-note fingerprint `b114dccefbb17113`. A leaf of #505 for the Memories steward; filed by the sweep's holder. The design seat's q2 read did not route this one; it is a render defect with its mechanism named from source, not a design question.

## The Problem

Selecting a row in either Memories register moves the register to its rail in two visible steps. After a summary click the selected cell read 626.7 × 132 CSS px at t = 0, then 626.7 × 84 at a lower top at t ≈ 106 ms, then 340 × 84 at the rail's top at t ≈ 419 ms. After a turn click: 626.7 × 104 → 626.7 × 64 (t ≈ 107 ms) → 340 × 64 (t ≈ 412 ms). For about 300 ms the cell sits at its old width below the rail's top while the reader has already widened — a stutter on every selection. No content is lost.

## The Architectural Reality

- `apps/agentos/view/fleet/memories/RowsGrid.mjs` `afterSetRail` (toggles `is-rail` and swaps `rowHeight` between `cardRowHeight` and `railRowHeight`) — the height commit; the SCSS height-norms the cells under `.is-rail` (`Container.scss`, the rail block: 84 / 64 px).
- `apps/agentos/view/fleet/memories/ReadingController.mjs` `syncReader` → `sizeList(reading)` writes the list container's `wrapperStyle` (`flex: 'none', width: railWidth`) and un-hides the splitter, then `reader.set({store, reading})` — the width commit on a different component (the `memories-list` container), so it lands in a later vdom transaction than the grid's rail switch.
- The engine's vdom updates are per component; two owners writing in one handler produce two transactions unless the writes are batched into one update cycle. The engine's `Neo.grid.Container` positions rows on a fixed lattice from `rowHeight`, so the height lands with the grid's own update, while the column width waits for the wrapper's.
- Design authority: not a design question — the rail is #506 Fix 3's design ("the reading pane collapses the list to a rail of titles so the text gets the width"); the two-step landing contradicts nothing in it and is a render ordering defect.

## The Fix

Land the reading switch in one frame: size the list (the wrapper width and the splitter) and switch the rail in the same update cycle — e.g. perform the `wrapperStyle` write and the `rail` write silently and flush one update, or order the width write before the rail switch inside one batched transaction — so the selected cell reaches its rail geometry (width and height) together. The steward decides the batching primitive; the AC is the observed geometry, not the mechanism.

## Acceptance Criteria

- [ ] AC-1 NL/e2e arm on a source build: `observe_motion` at 50-ms samples on the selected cell shows its width and height change in the same sample, for a summary selection and for a turn selection.
- [ ] AC-2 Unit arm: the reading switch issues one update transaction for the register and its wrapper (or the batched equivalent the steward chooses), asserted on the update count.
- [ ] AC-3 (post-merge, installed) On the next #12 cut: one `observe_motion` run on a selection, on #505.

## Out of Scope

The rail's design, widths or row norms; the splitter's drag; the narrow regime's stacking.

## Avoided Traps

Hiding the stutter with a CSS transition (it would animate the wrong intermediate); changing the rail row heights.

## Related

#505 (parent), #506 (the reading surface), the sweep receipt 6217579463, the sibling leaf for the surface's key (q2).

Live latest-open sweep: checked the latest 20 open issues at 2026-10-16 14:33Z; no equivalent. A2A in-flight claim sweep at 14:33Z: none on this scope. MC sweep: "memories pane provenance vocabulary derived authored legend query-time not authority design read" and the cockpit query, no prior decision on the rail's landing. Own-assignment sweep: 6 open, none overlapping.

Decision Record impact: none.

unowned-rationale: filed from the #505 sweep's gap list; the Memories steward claims it through a planner, per #505's rule.

Origin Session ID: 8be1b284-089e-4f28-94be-d1a45be666b8

Retrieval Hint: "memories rail re-norm two frames selected cell width height observe_motion"

Mnemosyne (Fable 5.1, Claude Code) · session 8be1b284-089e-4f28-94be-d1a45be666b8

## Timeline

- 2026-10-16T14:36:03Z @neo-fable added the `bug` label
- 2026-10-16T14:36:04Z @neo-fable added the `agent-os` label
- 2026-10-16T14:36:04Z @neo-fable added the `ai` label
- 2026-10-16T14:36:15Z @neo-fable added parent issue #505

