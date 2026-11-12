---
id: 942
title: 'The product bar spaces nothing: Nightshift touches the theme switch'
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-12T15:04:46Z'
updatedAt: '2026-11-12T15:04:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/942'
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
blockedBy: []
blocking: []
---
# The product bar spaces nothing: Nightshift touches the theme switch

## Context

The operator, 2026-11-12, with a screenshot of the installed FM: there is no gap between the Nightshift toggle and the theme switch. Measured on that installed candidate over the Neural Link (Institution `2ca27933`, Engine `6f15b707cf`, Brain `cfb998af`; dark theme; window 1981 CSS px wide):

| Pair | Left item's right edge | Right item's left edge | Gap |
|---|---|---|---|
| Nightshift toggle → theme switch | 1929 | 1929 | 0 px |
| "Agent OS" title → instance switcher | 199.375 | 199.375 | 0 px |

Both buttons compute `margin: 0`, and the toolbar computes `column-gap: normal`. The instance switcher's 1 px border meets the title's last glyph. The setup line (`agent-setup-progress`, shown only on a packaged first run) has no style rule at all, so it meets the switcher the same way. That third pair is read from source, since the line is hidden in this profile.

## The Problem

The product bar has no spacing rule. Its only spacing is the logo's own `margin-right`. The engine's toolbar zeroes button margins and declares no gap (`neo.mjs` `resources/scss/src/toolbar/Base.scss`: `.neo-toolbar .neo-button { margin: 0 }`). Until #929 the bar's right side held one control, so nothing showed. The Nightshift toggle is its first neighbour, and it touches. The next control added to the bar would collide the same way.

## The Architectural Reality

Design authority: "Action toolbars wrap so long-text buttons space cleanly instead of colliding." That is the shell's own rule for a row of agent buttons (`gap: var(--fm-space-2)`, `resources/scss/src/apps/agentos/Viewport.scss:447`). The header IA (`apps/agentos/design/institution-header-detail-ia.html`) draws its chrome bars with `gap: var(--sp-2)` (8 px) and lists the toggle and the switch as two controls of the product bar row. `apps/agentos/TOKENS.md` §04 puts spacing on the 4/8/12/16 rhythm.

- The bar is the viewport's top toolbar (`apps/agentos/view/Viewport.mjs`), styled by `.agent-top-toolbar` at `Viewport.scss:303`. Items: logo · title · instance switcher · setup line · spacer (`'->'`) · Nightshift toggle · theme switch.
- The logo keeps 15 px to the title through its `margin-right`, and 8 px at ≤ 560 px, where the title hides.
- The hidden setup line has no box (its rect reads 0 × 0 at the origin), so a flex gap does not count it.
- Re-valuing `.agent-button` is #208's avoided trap: it reaches the forms' buttons, which are not chrome. The rule belongs on the band.
- `Viewport.scss` is a stamped input of the Darwin visual baselines (`test/playwright/visual/__screenshots__/baseline-inputs.txt`). No golden frames the product bar (they frame `.fm-fleet-cockpit` and its parts), so the expected result is no pixel change plus a new stamp.

## The Fix

In `resources/scss/src/apps/agentos/Viewport.scss`:

1. `.agent-top-toolbar` gets `gap: var(--fm-space-2)`.
2. `.agent-logo`'s `margin-right` shrinks by the same gap, so the mark keeps its distance to the next item: 15 px → 7 px, and 8 px → 0 at ≤ 560 px.

## Acceptance Criteria

- [ ] Nightshift toggle → theme switch: 8 px between their borders, in both themes, both wide and at ≤ 560 px (glyph only).
- [ ] "Agent OS" title → instance switcher: 8 px.
- [ ] Logo → title stays 15 px wide, and logo → instance switcher stays 8 px at ≤ 560 px.
- [ ] `test/playwright/e2e/agentos/NightshiftToggleNL.spec.mjs` asserts the first criterion from DOM rects. It fails on `dev` and passes with the fix.
- [ ] `npm run check-visual-baselines` passes: the goldens, re-captured on macOS, show no change, and the stamp is rewritten (`npm run stamp-visual-baselines`). A maintainer can run the macOS capture for a contributor's PR, as on PR #938.
- [ ] The PR shows the product bar before and after, both wide and narrow.

## Out of Scope

- The logo's 15 px, which is off the rhythm. It stays as it is.
- The instance switcher's chip geometry, the band's height, and the engine toolbar.
- The cockpit bar (`ControlToolbar`), which already spaces its actions.

## Related

#929 and PR #933 (the toggle) · #208 (the band's control scale) · #197 (the band's right edge) · #505 (the design-sweep receipt for this read) · PR #938 (a stamp-only golden change).

Live latest-open + closed sweep: latest 20 open (59 open, red band; no owning ticket or epic for the product bar's spacing; `#13` is about token consumption) + closed search (`product bar gap`, `theme switch`, `top toolbar spacing`, `Nightshift button`; Out of Scope and Avoided Traps read on #197 and #208) at 2026-11-12T15:03Z; no equivalent or prior exclusion found.
A2A in-flight claim sweep: latest 30 messages at 15:03Z; no claim on this surface.
MC sweep: "product bar buttons touch, no gap between Nightshift toggle and theme switch, top toolbar spacing", 5 results, no prior decision.
Own-assignment sweep: 4 open (#732, #651, #649, #485), none overlapping.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: a7d339f7-fb5f-4f95-8fbf-31b109ff42e2
Retrieval Hint: "product bar no gap Nightshift toggle theme switch top toolbar spacing"

— Vega (Claude Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-11-12T15:04:47Z @neo-opus-vega added the `bug` label
- 2026-11-12T15:04:48Z @neo-opus-vega added the `good first issue` label
- 2026-11-12T15:04:48Z @neo-opus-vega added the `help wanted` label
- 2026-11-12T15:04:48Z @neo-opus-vega added the `agent-os` label
- 2026-11-12T15:04:48Z @neo-opus-vega added the `ai` label
- 2026-11-12T15:04:49Z @neo-opus-vega added the `design` label
- 2026-11-12T15:05:01Z @neo-opus-vega cross-referenced by #505

