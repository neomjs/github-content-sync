---
id: 199
title: 'The cockpit bar''s buttons are 48 px tall: the engine''s touch scale in a chrome row'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T11:59:40Z'
updatedAt: '2026-09-25T12:40:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/199'
author: neo-fable-clio
commentsCount: 0
parentIssue: 13
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T12:40:55Z'
---
# The cockpit bar's buttons are 48 px tall: the engine's touch scale in a chrome row

## Context

Design pass on #13, leaf 2 (the operator's "there are still some neo default-theme overrides", 2026-09-25). Measured on `dev@d2d9f86` + #198 in the browser (Playwright, CSS px, 1400 wide):

| element | height | why |
|---|---|---|
| preset buttons Overview / Focus / Review (`.fm-preset-button`) | 48 | the engine's `--button-height: 48px` (`--cmp-button-height`), `--button-padding: 0 16px`, `--button-min-width: 48px` |
| Reconnect, Start fleet | 48 | same |
| the cockpit bar (`.fm-cockpit-bar`) | 60 | 48 + 6 px padding top and bottom |
| the top toolbar / theme switch | 50 / 48 | same engine scale, one row |
| the pane header rows below (FLEET ×, ACTIVITY …) | ≈ 40 | FM chrome: `--fm-text-chrome` 11px/1 mono uppercase |
| the roster's sort chips | 19 | FM pills, 10px mono |

## The Problem

The bar is a chrome row between the top toolbar and the pane headers, but its controls carry the engine's touch scale: 48 px boxes with 16 px side padding, 1.2× the pane header rows and 2.5× the chips. The FM restyles these buttons in 191 declarations (Grace's census on #13) — colour, border, radius, weight, transition — and never the one property that sets their size, so the row reads as five oversized boxes in a dense instrument panel. The perspective switch's underline-vs-box noun (leaf 3) and the button-family colour conversion (later leaves) both sit on top of this scale.

## The Architectural Reality

- The engine's button reads its scale from variables at the element (`neo.mjs/resources/scss/src/button/Base.scss:22-28,94,106-107`: `height: var(--button-height)`, `padding: var(--button-padding)`, `min-width: var(--button-min-width)`, `border-radius: var(--button-border-radius)`, `font-size: var(--button-text-font-size)`), and the dark/light themes bind them to the core tokens. A re-valuation on the bar's scope renders because the engine's own rule reads the var on that element — Grace's caveat (the menu rebind that never rendered) does not apply here; the render check in both themes still runs.
- `cockpit/Container.scss:122-171` sets `border-radius: 6px` on the preset, Reconnect and Start fleet buttons by selector; with `--button-border-radius` re-valued on the bar those three lines are redundant.
- The bar's container queries (`:281` mark regime, `:308` vessel-narrow wrap) touch labels and wrap, not size; the 570 px wrap still holds at the new height.

## The Fix

`.fm-cockpit-bar` re-values the engine's scale variables for everything inside it: `--button-height: 32px` (the chrome row's control height, in the 4/8/12/16 rhythm as 2 × 16), `--button-min-width: 32px`, `--button-padding: 0 var(--fm-space-3)`, `--button-text-font-size: 13px`, `--button-border-radius: 6px` — and the three per-selector `border-radius: 6px` lines go. `--button-height` also drives the bar's text-only presets and the glyph-only toggles alike (the engine's `min-width` keeps a glyph button square). The visual goldens that show the bar and everything under it move (a vertical shift); re-rendered from a full run alone with a by-eye read.

**Measured after the fix (2026-09-25 12:05Z):** the buttons are 32 px; the bar settles at 53 px, not 44 — its height is now set by the two-row wake telltale (41 px), which is the next leaf on #13, not this one.

## Acceptance Criteria

- [ ] `.fm-preset-button`, `.fm-reconnect-button`, `.fm-fleet-start` and the bar's toggles measure 32 px tall at 1400 wide (probe as above); the bar's own height is whatever its tallest remaining child sets (the telltale, leaf 3).
- [ ] `cockpit/Container.scss` carries no per-selector `border-radius` for those buttons; the bar scope carries the five `--button-*` re-valuations, each commented as the engine property it drives.
- [ ] Both themes render the re-valuation (light + dark captures in the visual suite; the mark-regime container query at 760 px and the vessel-narrow wrap at 570 px keep their asserted geometry).
- [ ] `npm run test-visual` alone: every moved golden updated in this leaf and named; the baseline stamp re-issued; `check-visual-baselines` 0.

## Out of Scope

- The preset switch's noun and grouping (segmented control) and the wake telltale's two-row stack — leaf 3.
- Colour / border / state conversions of the button family to `--button-*` (Grace's 102) — later leaves, one family each.
- The top toolbar's own scale (its 50 px band and the theme switch) — the app-shell row, judged separately.

## Related

#13 (parent) · #197 / #198 (leaf 1: the frame gutter) · the design pass read (issuecomment-5831663831) · Grace's census (issuecomment-5831294609)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 12:00Z; no equivalent. A2A claim sweep: no claim on the bar's scale. Memory Core: no prior decision on the bar's control height.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm cockpit bar button height 32 --button-height re-valuation chrome scale"

## Timeline

- 2026-09-25T11:59:40Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T11:59:41Z @neo-fable-clio added the `bug` label
- 2026-09-25T11:59:41Z @neo-fable-clio added the `ai` label
- 2026-09-25T11:59:41Z @neo-fable-clio added the `design` label
- 2026-09-25T12:00:07Z @neo-fable-clio added parent issue #13
- 2026-09-25T12:04:00Z @neo-fable-clio cross-referenced by PR #200
- 2026-09-25T12:20:01Z @neo-fable-clio cross-referenced by #201
- 2026-09-25T12:40:55Z @tobiu referenced in commit `717688c` - "Merge pull request #200 from neomjs/fix/199-cockpit-bar-scale

fix(cockpit): the cockpit bar's buttons sit on the chrome scale, not the engine's touch scale (#199)"
- 2026-09-25T12:40:55Z @tobiu closed this issue
- 2026-09-25T13:10:52Z @neo-fable-clio cross-referenced by #203
- 2026-09-25T13:42:39Z @neo-fable-clio cross-referenced by #206
- 2026-09-25T13:56:26Z @neo-fable-clio cross-referenced by #208
- 2026-09-25T14:00:11Z @neo-fable-clio cross-referenced by PR #209

