---
id: 208
title: The theme switch is a 48 px touch box in the 50 px shell band
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T13:56:25Z'
updatedAt: '2026-09-25T14:48:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/208'
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
closedAt: '2026-09-25T14:48:12Z'
---
# The theme switch is a 48 px touch box in the 50 px shell band

## Context

Design pass on #13, leaf 6 — the app-shell row, judged separately from the cockpit bar (#199's out-of-scope line). Measured on `dev@893ec3b` + #207 at 1400 wide (Playwright, CSS px):

| element | box | note |
|---|---|---|
| `.agent-top-toolbar` | 1400 × 50 | `min-height: 50px`, padding `0 var(--fm-frame-gutter)` |
| logo (`.agent-logo img`) | 30 × 30 | top 10 |
| wordmark (`.agent-shell-title`) | 69 × 18 | |
| instance switcher (`.fm-instance-trigger.neo-button`) | 200 × 21 | its own chip geometry, padding 4 / 8 |
| theme switch (`.agent-theme-button.neo-button`) | **49 × 48**, top 1 | `--button-height: 48px`, `--button-padding: 0 16px`, glyph only |
| the cockpit bar's controls, one row below | 32 tall | #200 |

## The Problem

The band's largest element is its least important control. The theme toggle is a 48 px box in a 50 px band — 1 px of air above and below — heavier than the logo (30) and the instance scope chip (21) beside it, and 1.5× the bar's controls directly underneath. It is the last control in the chrome on the engine's touch scale; it reads as a tile in the corner, not as a quiet switch.

## The Architectural Reality

- `.agent-top-toolbar` (`resources/scss/src/apps/agentos/Viewport.scss:201-212`) declares the band's ground, `min-height` and gutter padding — no control scale. `.neo-button.agent-button` (`:247-266`) is the hierarchy (border, ground, ink, weight) shared with other shell buttons (forms, `agent-submit-button`), so the scale does not belong on that class.
- The engine's button reads `--button-height`, `--button-min-width`, `--button-padding` and `--button-border-radius` at the element (`neo.mjs/resources/scss/src/button/Base.scss`), the theme binds them on a zero-specificity ancestor; a re-valuation on the band's scope sizes the switch through the engine's own rule — the #199 mechanism, verified there.
- The instance switcher is a `.neo-button` with its own chip geometry (21 px) — it must measure the same after the scope re-valuation (probe).
- The glyph-only button's empty text span is already zero-width (`Viewport.scss:125-130`), so `--button-padding: 0` with `--button-min-width: 32px` gives a 32 × 32 square with the glyph centered.
- Goldens: the viewport captures carry the band (`cockpit-vessel-314`, `cockpit-intermediate-720`); the cockpit-element captures (`cockpit-default-shell`, `cockpit-review-1280`) start below it and stay.

## The Fix

`.agent-top-toolbar` re-values the four scale vars: `--button-height: 32px`, `--button-min-width: 32px`, `--button-padding: 0`, `--button-border-radius: 6px` — the switch becomes a 32 × 32 square centered in the 50 px band (9 px above and below), its right edge on the gutter line as before; each var commented as the engine property it drives (the #199 shape). Nothing on `.agent-button`, nothing on the band's height. Goldens that carry the band re-render from a full run alone; stamp re-issued.

## Acceptance Criteria

- [ ] At 1400 and 600 wide the theme switch measures 32 × 32 with its top at 9 inside the 50 px band and its right edge on the gutter line (1380 at 1400); the instance switcher still measures 21 px tall; logo and wordmark unchanged (probe).
- [ ] The diff touches only `.agent-top-toolbar`'s scope: four `--button-*` re-valuations, each commented.
- [ ] `npm run test-visual` alone: every moved golden updated and named (the band-bearing captures); the stamp re-issued; `check-visual-baselines` 0.

## Out of Scope

- The band's 50 px height and the logo's 30 px (the shell's identity row keeps its measure).
- The instance switcher's chip geometry.
- Shell buttons outside the band (accounts, forms) — their scale is judged where they live.

## Avoided Traps

- Re-valuing on `.agent-button`: it reaches the forms' buttons, which are not chrome.
- Shrinking the band to 44 to match the bar: the logo needs its 30 px and the band is the identity row, not a control row.

## Related

#13 (parent) · #199 / #200 (the bar's scale, the same mechanism) · #197 / #198 (the gutter both rows end on) · Grace's census (issuecomment-5831294609)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 13:55Z; no equivalent. A2A claim sweep (last 60 min, all read-states, 13:54Z): no claim on the shell band. Memory Core: #199's chrome-scale decision applies; no prior decision on the band's controls. Own-assignment sweep: #206 (leaf 5, at review) is the only same-surface hit.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm shell band top toolbar theme switch 48px touch scale --button-height 32 agent-top-toolbar"

## Timeline

- 2026-09-25T13:56:25Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T13:56:26Z @neo-fable-clio added the `bug` label
- 2026-09-25T13:56:26Z @neo-fable-clio added the `ai` label
- 2026-09-25T13:56:26Z @neo-fable-clio added the `design` label
- 2026-09-25T13:56:34Z @neo-fable-clio added parent issue #13
- 2026-09-25T14:00:11Z @neo-fable-clio cross-referenced by PR #209
- 2026-09-25T14:48:12Z @tobiu closed this issue

