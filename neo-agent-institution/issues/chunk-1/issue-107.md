---
id: 107
title: 'The instance pill spills, the state word clips, the light title vanishes'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T17:33:02Z'
updatedAt: '2026-09-04T22:14:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/107'
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
closedAt: '2026-09-04T22:14:19Z'
---
# The instance pill spills, the state word clips, the light title vanishes

## Context

The cockpit design pass of 2026-09-01 left a residual ledger on Epic #10 ([comment 5501089367](https://github.com/neomjs/neo-agent-institution/issues/10#issuecomment-5501089367)): four defects read off re-captured goldens, disclosed there because a re-captured golden blesses what it shows (Ada's RA-1 on PR #71) and the defect loses its only observer at merge. Item 4 (the 2px-taller synthesis golden) resolved on 2026-09-04: the cockpit's own `--tab-button-height: 30px` finally outranks the theme's 32px inline header once neomjs/neo#18145 took selectors out of the theme value files, and the goldens were re-captured with that read in PR #93 (`45211e8`). Items 1–3 are this ticket, re-measured on `dev@ce47818` (engine pin 6, built CSS fresh) with an untracked Playwright probe at 314×900 and 1280×720 in both skins:

| Surface | 314 px vessel | 1280 px control |
|---|---|---|
| wordmark "Agent OS" box | x 57–126 | x 65–134 |
| instance switcher box | x 126–254 (128 px) | x 134–334 |
| switcher LABEL box (`127.0.0.1:8083/fleet`) | x 115–265 (150 px) — 11 px over the wordmark's "OS", 11 px past the button's right edge | x 159–309, inside |
| roster card outer / identity column | 191 px / 53 px | 573 px / 413 px |
| state word `unobserved` (60 px) | right edge 201 vs state-line right 178 → 23 px past the line, 15 px under the verbs (left edge 186) | inside |
| wordmark ink vs rail, dark skin | rgb(240,242,240) on rgb(14,19,26) = 16.56:1 | same |
| wordmark ink vs rail, light skin | rgb(240,242,240) on rgb(233,237,243) = **1.04:1** | same |

`cockpit-vessel-314.png` (re-captured 2026-09-04 in PR #98) holds the first two as expected pixels.

## The Problem

1. **The instance pill spills over the wordmark and past the theme button.** `.fm-instance-label` declares `max-width: 180px; overflow: hidden; text-overflow: ellipsis`, but it sits in `.neo-button-text` (a flex row) with the flex default `min-width: auto`, so its minimum is its own nowrap text (150 px). The button box shrinks to 128 px, the text row cannot, and the centred overflow paints 11 px on each side — over the "OS" on the left, past the switch-theme button on the right. The ellipsis never fires.
2. **The roster's state word slides under its verbs.** In the sub-narrow card band (≤240 px content — the one-column roster inside the 314 px vessel hands each card 161 px), the identity column keeps ~53 px beside the two 32 px verbs. `.fm-card-state` is `flex: none; white-space: nowrap` with no containment, so a 60 px word overflows the state line and the later-painted verbs cover it: "unobse". The #23 law — *no mid-word clipping anywhere; full truth stays reachable via T5 titles* — is broken on the product's densest card.
3. **The light skin cannot read the wordmark.** The shell title is a stock `label` with the engine's label ink in both skins (rgb 240,242,240); the top band paints `--fm-rail` (#0e131a dark, #e9edf3 light). 16.56:1 in the dark skin, 1.04:1 in the light one — the wordmark is invisible in light.

## The Architectural Reality

- Shell chrome: `apps/agentos/view/Viewport.mjs:84-113` — `agent-top-toolbar` (logo · plain `label` "Agent OS" · `InstanceSwitcher` · `->` · theme button); `resources/scss/src/apps/agentos/Viewport.scss:193-219` (band, 560 px padding step, logo). No rule addresses the label; the SSOT plan (`apps/agentos/design/fleet-manager-cockpit-plan.html`) specifies no shell wordmark.
- Instance switcher skin: `resources/scss/src/apps/agentos/fleet/instances/SwitcherButton.scss:6-42` — trigger, `.neo-button-text`, `.fm-instance-label`. The trigger's accessible name and `title` already carry the full `<label> — <state word>` (`SwitcherButton.mjs:263-264`), so an end-ellipsis on the label loses no truth.
- Roster card: `resources/scss/src/apps/agentos/fleet/roster/card/Container.scss` — card-owned width modes via `@container` (ADR 0029): regular, narrow ≤289 px (44 px verbs, engine hidden), sub-narrow ≤240 px (24 px avatar, 32 px verbs, name floor ~50 px) at lines 365-418; `.fm-card-state-line` :148, `.fm-card-state` :156 (`--fm-ink-dim`, `fm-state-hot` weight — the WCAG 1.4.1 text channel, never a hue), verbs :222-233. The word is the visible state channel, so it may move but never disappear.
- Precedent for the collapse shape: #23's declared order (wide → mid → narrow marks with T5 titles) on the cockpit bar, and the 720-band witness in `test/playwright/visual/FleetCockpitVisual.spec.mjs:192`.
- Type: the wordmark renders the engine's 16 px label font; the §04 roles (`apps/agentos/TOKENS.md:51`) have no chrome-title role. Colour binds to the ink tier (`--fm-ink` measured ≥4.5:1 on every surface in both skins per TOKENS.md).

## The Fix

**Shell header** (`Viewport.mjs` + `Viewport.scss` + `SwitcherButton.scss`):
- The wordmark label gains `cls: ['agent-shell-title']`; the rule binds `color: var(--fm-ink)` and `flex: none` (the box never shrinks below its word).
- The switcher trigger contains its text: `max-width: 100%; overflow: hidden` on the trigger, `min-width: 0; max-width: 100%` on `.neo-button-text`, `flex: 0 1 auto; min-width: 0` on `.fm-instance-label` — the label now ellipsizes at its END inside the button, and the button never paints outside its box at any width.
- At the vessel band (the existing `max-width: 560px` step) the wordmark yields: `display: none`. The logo is the product mark, the window title already reads `<label> — Agent OS` (`VesselContainer.mjs:828`), and the instance scope keeps its full label — scope information outranks a redundant word when only one fits. Recorded design decision; a delta is a comment on this ticket, never silent drift.

**Roster card** (`card/Container.scss`):
- Sub-narrow band (≤240 px content): the head wraps (`flex-wrap: wrap`) and the verbs take their own full-width row, right-aligned (`flex-basis: 100%; justify-content: flex-end`). The identity column grows from ~53 px to ~129 px; dot + word (76 px) fit whole.
- In the same band `.fm-card-state-line` gains `flex-wrap: wrap; row-gap: var(--fm-space-1)`: it engages only under pressure, so the lane badge / presence band / telltale wrap to a second row instead of overflowing; the badge's `margin-left: auto` keeps it right-pinned on its row. No text node is ever cut. *(Scoped to the sub-narrow band on 2026-09-04 during implementation: applied in every band, the wrap overflowed the narrow band's height-constrained card wall by 19 px in the `AgentCardSynthesisRenderNL` fixture at 294 px — the narrow band stays out of scope, its goldens byte-identical.)*

**Witnesses** (`FleetCockpitVisual.spec.mjs`): the 314 arm asserts (a) the switcher's label box lies inside the switcher box and the wordmark/switcher boxes do not intersect, (b) every card's state-word right edge ≤ its state-line right edge and no intersection with its verbs; a light-skin header arm asserts the wordmark's computed contrast against the band ≥ 4.5:1. `cockpit-vessel-314.png` is re-captured with the design read in the PR body (cards one row taller, header clean); the synthesis goldens (294 outer = 264 content, the 44 px narrow band) must come back byte-identical or the diff is read and named. The visual baseline stamp is regenerated (SCSS is a stamp input).

## Contract Ledger Matrix

None — no consumed surface changes: the switcher's accessible name/`title`, the card's state vocabulary and the class names stay as they are; the change is geometry and one colour binding.

## Decision Record impact

`none` — aligned-with ADR 0029 (card-owned responsiveness via `@container`; the shell band uses the viewport axis because the top chrome IS the viewport's width).

## Acceptance Criteria

- [ ] AC-1: at 314×900 the switcher label's box is inside the switcher's box, the wordmark and switcher boxes do not intersect, and the theme button is not overpainted — computed receipts in the 314 visual arm.
- [ ] AC-2: at 314×900 every roster card's state word right edge ≤ its state-line right edge, and no state word intersects its verbs; the RA-1 identity floor (≥44 px) holds — computed receipts in the same arm.
- [ ] AC-3: the wordmark's computed contrast against the top band is ≥ 4.5:1 in BOTH skins — computed receipt in a light-skin arm.
- [ ] AC-4: the vessel-band decision (wordmark yields ≤560 px, instance label keeps its tail) is recorded in the PR body with a before/after read of `cockpit-vessel-314.png`; the golden is re-captured under the visual config and read by eye.
- [ ] AC-5: the synthesis goldens (`AgentCardSynthesisRenderNL`, both skins × six widths) are byte-identical, or every changed frame is named and read in the PR body.
- [ ] AC-6: `npm run test-visual`, `npm run test-unit` and the cockpit component suite green; `check-visual-baselines` green on the pushed head.

## Out of Scope

- A chrome-title type role or resizing the wordmark (a §04 role decision — the ledger names the colour only).
- The 240–289 px narrow band's 44 px verb pair and the regular band's anatomy.
- The state vocabulary, the dot's colour channel, and the presence/telltale semantics.
- Ledger item 4 (resolved in PR #93) and the light-skin goldens beyond the header arm.

## Avoided Traps

- **Ellipsis on the state word** (`overflow: hidden; text-overflow: ellipsis`): "unobs…" is still a cut word — the #23 law forbids it, and the word is the WCAG 1.4.1 text channel. Layout grows; text stays whole.
- **Hiding the state word at sub-narrow** with a `title` on the dot: recreates the hue-only carrier TOKENS.md's #14619 audit already names on the product's densest band.
- **`max-width` on the switcher label alone**: the existing 180 px cap proves a cap without `min-width: 0` on a flex item does nothing.
- **A viewport media query on the card**: the card owns its width (ADR 0029); only the shell band uses the viewport axis.

## Related

Epic #10 (parent — the ledger comment) · #23 (the collapse-order law and its computed-receipt bar) · PR #71 / PR #72 (where the goldens and the light receipt were read) · PR #93 (`45211e8`, item 4) · PR #98 (`450e26a`, the current 314 golden) · #17264 / #17265 (the §04 chip and type passes) · TOKENS.md §04 · ADR 0029.

Live latest-open sweep: latest 20 open issues read at 2026-09-04T17:31Z — none equivalent (#103 is the FLIP-settle hold, #23 is closed). A2A claim sweep: last 30 messages, all read-states, 17:31Z — no claim on the shell header or the roster card. MC sweep: `query_raw_memories` on the pill/wordmark overlap and on the clipped state word — only the 2026-09-01 ledger and its origin reads, no prior decision; item 4's cause found there (the 30 px tab header). Own-assignment sweep: #103, #87, #10 open — the ledger on #10 IS this ticket's source, no overlap otherwise.

Origin Session ID: 46962d8b-08f3-49a3-8049-d74e2052af37

Retrieval Hint: `query_raw_memories("instance pill spills over the wordmark at vessel width, state word slides under the verbs, light shell title contrast")`


## Timeline

- 2026-09-04T17:33:02Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T17:33:04Z @neo-fable-clio added the `bug` label
- 2026-09-04T17:33:04Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T17:33:05Z @neo-fable-clio added the `ai` label
- 2026-09-04T17:33:05Z @neo-fable-clio added the `design` label
- 2026-09-04T17:35:39Z @neo-fable-clio cross-referenced by #10
- 2026-09-04T17:52:02Z @neo-fable-clio cross-referenced by PR #108
- 2026-09-04T22:14:19Z @tobiu referenced in commit `28bc313` - "Merge pull request #108 from neomjs/agent/107-shell-title-state-word

fix(agentos): the scope control contains its label, the sub-narrow card wraps its verbs, the wordmark reads in both skins (#107)"
- 2026-09-04T22:14:19Z @tobiu closed this issue

