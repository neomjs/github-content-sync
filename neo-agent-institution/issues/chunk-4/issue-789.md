---
id: 789
title: A selected Memories row draws a hard double ring where the roster draws a soft tint
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-23T15:12:17Z'
updatedAt: '2026-10-23T22:16:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/789'
author: neo-fable-clio
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
closedAt: '2026-10-23T22:16:22Z'
---
# A selected Memories row draws a hard double ring where the roster draws a soft tint

## Context

Operator report (2026-10-23, on his installed Fleet Manager): selected items look odd — thicker top and bottom edges than sides, and a color that does not fit. The only cockpit surface that draws a selection ring of its own is the Memories grid; code read at `origin/dev` 024a7a5586.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-23 14:5xZ; no equivalent (#715, closed, fixed the row's re-norm, not its selection paint). A2A claim sweep (last 14 broadcasts): none. Memory Core rationale sweep: no decision on the Memories selection paint recovered. Own-assignment sweep: #505/#507 are the parents.

## The Problem

`resources/scss/src/apps/agentos/fleet/memories/Container.scss:170-173`: a selected row's card gets `border-color: var(--fm-signal)` on its existing 1 px border (`:46`) **and** `box-shadow: inset 0 0 0 1px var(--fm-signal)` — two mechanisms, a 2 px hard ring in the full signal color. The cockpit's selection language is the roster card's (`resources/scss/src/apps/agentos/fleet/roster/Container.scss:134-137`): background `color-mix(in srgb, var(--fm-signal) 8%, var(--fm-panel))`, border `color-mix(in srgb, var(--fm-signal) 24%, var(--fm-line-soft))`, `box-shadow: var(--fm-shadow-selection)` — a tint, not a ring. The Memories row diverges from it, which is the "not a fitting color".

The uneven edges did not reproduce in the visual fixture (build finding, 2026-10-23): at DPR 1 and 2 the old ring rendered a uniform 2 px on all four sides, the card sitting 4 px inside its cell with nothing clipping it. The operator's build or width may differ; with the ring gone the symptom has no surface left.

## The Architectural Reality

- `memories/Container.scss:160-166` already zeroes the engine's grid selection tokens (`--grid-rowmodel-selected-cell-background-color: transparent`, cell color inherit) and paints the selection on the card instead — the right place; only the paint is wrong.
- The roster's rule is the cockpit's selection language.
- Build finding: the grid cell's `overflow: hidden` clips the roster's outer shadow (`0 6px 16px`) into faint side halos cut flat at the card's bottom; a grid row is not a card on a surface, so the shadow is not part of the language here.
- A turn row carries an authored 2 px signal stripe on its left border; the 24 % border mix must not repaint it.

## The Fix

Spec-only SCSS: the selected Memories card and turn (`.fm-memories-card`, `.fm-memories-turn`) adopt the roster card's tint and border (8 % signal tint, 24 % signal border mix), with `box-shadow: none` explicitly (no inset ring, no outer shadow); the turn keeps `border-left-color: var(--fm-signal)` for its authored stripe.

Design authority: the roster card's selection language (`roster/Container.scss:134-137`) is the cockpit's, minus the shadow a clipped cell cannot carry; a surface-specific ring is not.

## Acceptance Criteria

- [ ] AC-1: goldens on the PR, light and dark, of a selected summary row (the visual arm `memories-row-selected-1280`): the four edges read the same; side by side with a selected roster card the tint and border match.
- [ ] AC-2: the selected card's computed `box-shadow` is `none` and its background and border-color equal a selected roster card's (computed-style reads in the arm); a selected turn keeps its 2 px signal left stripe.
- [ ] AC-3 (post-merge): the operator's installed app shows the same selection tint on a roster card and a Memories row.

## Out of Scope

Keyboard selection in the Memories grid (works today through the grid's row model); the reader's own frame; the Activity stream's rows (#790); the cell's overflow or the card's height (the shadow is dropped instead).

## Avoided Traps

Tuning the ring's color alone (keeps two mechanisms); overriding the engine grid tokens with a color (the card is the surface here, not the cell); forcing the outer shadow through a cell that clips it.

## Related

#505 (parent) · #506 (the reading pane) · #713 / #715 (the Memories surfaces' provenance key and rail re-norm) · #788 (the sibling splitter leaf; whichever PR lands second re-captures the shared Memories reading golden) · the roster card's selection rule.

handoff: Clio (design lead) drives it through a delegated builder; PR to follow.

Origin Session ID: d0801374-2558-4c56-97d0-ba4a719149a3

Handoff Retrieval Hints: `query_raw_memories("memories selected row ring signal roster card tint shadow-selection clipped cell")`; the operator's 2026-10-23 FM report.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session d0801374-2558-4c56-97d0-ba4a719149a3


## Timeline

- 2026-10-23T15:12:18Z @neo-fable-clio added the `bug` label
- 2026-10-23T15:12:18Z @neo-fable-clio added the `agent-os` label
- 2026-10-23T15:12:19Z @neo-fable-clio added the `ai` label
- 2026-10-23T15:12:19Z @neo-fable-clio added the `design` label
- 2026-10-23T15:13:38Z @neo-fable-clio added parent issue #505
- 2026-10-23T17:04:04Z @neo-fable-clio cross-referenced by PR #792
- 2026-10-23T17:04:07Z @neo-fable-clio cross-referenced by PR #793
- 2026-10-23T17:24:21Z @neo-fable-clio cross-referenced by #794
- 2026-10-23T22:06:31Z @neo-fable-clio referenced in commit `bdee5d5` - "fix(memories): a selected row wears the roster's selection paint (#789)

A selected Memories card drew a hard double ring: the signal border plus a 1px inset shadow in the
full signal color. It now wears the roster card's selection language: the 8% signal tint, the border
mixed 24% toward the signal, and --fm-shadow-selection, with no inset ring. A selected turn keeps its
2px signal stripe, which is its authored provenance rather than selection paint.

A visual arm lands three session summaries through a new Memories driver, selects one at 1280x1000 and
asserts the card casts the roster's --fm-shadow-selection without an inset layer in both skins, with a
golden of the list each."
- 2026-10-23T22:06:32Z @neo-fable-clio referenced in commit `e7ca6ee` - "fix(memories): the selected row keeps the tint and border; the cell cannot carry the shadow (#789)

The pooled cell's overflow clipped --fm-shadow-selection into faint side halos cut flat at the bottom,
and a grid row is not a card on a surface. A selected Memories card or turn keeps the roster card's tint
and border and now casts no shadow (box-shadow: none, explicit); a selected turn keeps its signal stripe.

The arm selects the resident's roster card beside the row and asserts, in both skins, that the row casts
no shadow while its background and border equal the selected roster card's. Both goldens regenerated."
- 2026-10-23T22:06:32Z @neo-fable-clio referenced in commit `127cf87` - "test(visual): the reading golden and the baseline stamp follow the selection paint (#789)

The Memories reading goldens from #788 show the selected card in the ring paint; with the row's tint and
border they are re-captured, dark and light. Re-stamped after the rebase onto dev at the engine its lock
pins (0ab667195e, moved by the all-deps bump after the last stamp)."
- 2026-10-23T22:16:22Z @tobiu referenced in commit `42df0c4` - "fix(memories): a selected row wears the roster's selection paint (#789) (#793)

* fix(memories): a selected row wears the roster's selection paint (#789)

A selected Memories card drew a hard double ring: the signal border plus a 1px inset shadow in the
full signal color. It now wears the roster card's selection language: the 8% signal tint, the border
mixed 24% toward the signal, and --fm-shadow-selection, with no inset ring. A selected turn keeps its
2px signal stripe, which is its authored provenance rather than selection paint.

A visual arm lands three session summaries through a new Memories driver, selects one at 1280x1000 and
asserts the card casts the roster's --fm-shadow-selection without an inset layer in both skins, with a
golden of the list each.

* fix(memories): the selected row keeps the tint and border; the cell cannot carry the shadow (#789)

The pooled cell's overflow clipped --fm-shadow-selection into faint side halos cut flat at the bottom,
and a grid row is not a card on a surface. A selected Memories card or turn keeps the roster card's tint
and border and now casts no shadow (box-shadow: none, explicit); a selected turn keeps its signal stripe.

The arm selects the resident's roster card beside the row and asserts, in both skins, that the row casts
no shadow while its background and border equal the selected roster card's. Both goldens regenerated.

* test(visual): the reading golden and the baseline stamp follow the selection paint (#789)

The Memories reading goldens from #788 show the selected card in the ring paint; with the row's tint and
border they are re-captured, dark and light. Re-stamped after the rebase onto dev at the engine its lock
pins (0ab667195e, moved by the all-deps bump after the last stamp)."
- 2026-10-23T22:16:23Z @tobiu closed this issue

