---
id: 203
title: 'Twelve font-size sites in the FM SCSS pick a pixel, not a §04 role'
state: CLOSED
labels:
  - enhancement
  - ai
  - design
  - tech-debt
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T13:10:51Z'
updatedAt: '2026-09-25T13:39:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/203'
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
closedAt: '2026-09-25T13:39:07Z'
---
# Twelve font-size sites in the FM SCSS pick a pixel, not a §04 role

## Context

Design pass on #13, leaf 4. Grace's note on #200 (review 5317515736): `--button-text-font-size: 13px` picks a pixel, and the §04 type ladder says "a surface picks a ROLE, never a pixel" (`Viewport.scss:31-44`; `apps/agentos/TOKENS.md:51`). The bar's 13 px is not alone. Census on `dev@717688c`:

```
grep -rn -E "font-size\s*:\s*[0-9.]+px" resources/scss/src/apps/agentos/
```

13 hits: 12 sites and one comment line (`fleet/cockpit/Container.scss:24`). The ladder: `--fm-text-display` 600 14px/1.3 sans · `--fm-text-body` 12px/1.45 sans · `--fm-text-detail` 11px/1.4 mono · `--fm-text-micro` 10px/1.3 mono · `--fm-text-chrome` 11px/1 mono (+ `.08em` tracking).

| site | value | what it types | nearest role | shift | disposition |
|---|---|---|---|---|---|
| `Viewport.scss:119` | 12px | tab-header toolbar action glyph | — (glyph scale, §04 T3) | — | record |
| `Viewport.scss:305` | 12px | agent-button glyph ("one step below the label") | — (glyph scale) | — | record |
| `Viewport.scss:401` | 12px mono, `.22em`, uppercase | welcome eyebrow | chrome 11px/1 | 1 px | by-eye read: fold or record |
| `Viewport.scss:418` | 17px/1.6 | welcome lede | none (the ladder tops at display 14) | — | record (prose above the ladder) |
| `Viewport.scss:439` | 15px/1.6 | placeholder prose | none | — | record |
| `accounts/Panel.scss:18` | 12px | selector button text | body 12px/1.45 | 0 | fold |
| `fleet/roster/card/Container.scss:75` | 14px, after `font: var(--fm-text-display)` (14px) | monogram | display | 0 | delete the redundant literal |
| `…/card/Container.scss:472` | 12px | monogram at the narrow band | — (letterform glyph) | — | record |
| `…/card/Container.scss:482` | 16px | card glyph | already a `§04 recorded exception (#17265)` | — | keep |
| `…/card/Container.scss:515` | 10px | "the micro floor" | micro 10px/1.3 mono | 0 | fold if the stack matches, else record |
| `…/card/Container.scss:521` | 13px | card action glyph, 32 × 32 | — (glyph scale) | — | record |
| `fleet/cockpit/Container.scss:109` | `--button-text-font-size: 13px` | the bar's button text (#199) | body 12px/1.45 | 1 px | by-eye read: fold via `.neo-button-text { font: var(--fm-text-body) }` on the bar's scope, or record |

## The Problem

The rule has two halves and the SCSS carries only one. The spacing rhythm grants "a value outside the rhythm is a recorded exception in its file's SCSS header — never silent" (`Viewport.scss:46-47`); the type ladder's own text names no escape hatch, and neo `#17265` (the §04 pass, closed) left these twelve behind — glyph sizes, two prose sizes above the ladder, two zero-shift folds nobody took, and two 1 px design calls. Silent literals are what the ladder exists to stop: a reader cannot tell a decision from drift.

## The Architectural Reality

- The roles are `font` shorthands (`Viewport.scss:38-42`), consumed as `font: var(--fm-text-*)`; a site that needs another weight re-declares `font-weight` after the role (`:33-36`). The engine's button reads `--button-text-font-size` as a bare size at `.neo-button-text` (`neo.mjs/resources/scss/src/button/Base.scss`), so a role there needs a scope rule on `.neo-button-text`, not a var re-valuation (Grace's note).
- Glyph sizes are the §04 T3 axis ("the glyph decorates, never carries"), not the type ladder; the card already records one as a `§04 recorded exception (#17265)` (`card/Container.scss:482`). A `--fm-glyph-*` token group would be a new token = a design decision per `TOKENS.md` "Adding a token" — out of this leaf.
- Grace's clause from the ladder's own spec review (neo PR `#17279`, 2026-08-17; Memory Core): nearest role; a shift of 1 px or more is a named exception in the SCSS header, mirroring the card's recorded-exception precedent. This leaf applies that clause to the residue.
- The visual goldens move only where a fold changes rendered text: the bar (`cockpit-default-shell`, `cockpit-review-1280`, the bar-composition e2e captures on darwin) if the 13 px folds; the welcome surface has no golden.

## The Fix

1. Folds with zero shift: `accounts/Panel.scss:18` → `font: var(--fm-text-body)` (weight re-declared after); `card/Container.scss:75` → delete the literal; `card/Container.scss:515` → `font: var(--fm-text-micro)` if its stack is mono, else recorded.
2. The two 1 px calls get a by-eye read at both sizes (captures in the PR): the bar's presets / Reconnect / Start fleet at body 12 px vs 13 px; the welcome eyebrow at chrome 11 px vs 12 px. The read decides fold or record; the PR body shows both.
3. Everything else becomes a recorded exception at the site (`/* §04 recorded exception: glyph scale */` or `prose above the ladder`), and `Viewport.scss`'s ladder comment gains the missing half-sentence: a type literal outside the ladder is a recorded exception in its file, never silent — the spacing rhythm's own rule.
4. Goldens that move re-render from a full run alone with a by-eye read; the stamp re-issued.

## Acceptance Criteria

- [ ] The census grep returns only lines that carry a `§04 recorded exception` comment or sit inside a comment; every fold in the table is a role shorthand with zero size shift.
- [ ] The two 1 px calls are decided by captures shown in the PR (both sizes), and the chosen disposition is written at the site.
- [ ] `Viewport.scss`'s type-ladder comment states the recorded-exception rule for type, as the rhythm comment does for spacing.
- [ ] `npm run test-visual` alone: every moved golden updated and named; the stamp re-issued; `check-visual-baselines` 0.

## Out of Scope

- A glyph-scale token group (a new token = a design decision, its own leaf).
- The preset switch as a segmented control (the next leaf).
- Colour, border and state conversions of the button family (Grace's 102).

## Avoided Traps

- Folding the 13 px and the eyebrow blind: a 1 px shift on the bar's only text controls is a design call, and the ladder's author already ruled that such shifts are named, not silent.
- Minting `--fm-text-lede` for two transient prose surfaces: a role for two sites is a token nobody else binds.

## Related

#13 (parent) · #199 / #200 (the 13 px origin) · Grace's note (review 5317515736) · neo `#17265` (the §04 pass) · neo PR `#17279` (the ladder spec + the exception clause)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 13:09Z; no equivalent. A2A claim sweep (last 60 min, all read-states, 13:08Z): no claim on the type ladder. Memory Core: Grace's 2026-08-17 spec review (the count of 68 literals, the exception clause) and her 12:20Z note — both applied above. Own-assignment sweep: #201 (leaf 3, at the merge gate) is the only same-surface hit.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm §04 type ladder pixel font-size literals recorded exception fold role census"

## Timeline

- 2026-09-25T13:10:51Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T13:10:52Z @neo-fable-clio added the `enhancement` label
- 2026-09-25T13:10:52Z @neo-fable-clio added the `ai` label
- 2026-09-25T13:10:53Z @neo-fable-clio added the `design` label
- 2026-09-25T13:10:53Z @neo-fable-clio added the `tech-debt` label
- 2026-09-25T13:11:02Z @neo-fable-clio added parent issue #13
- 2026-09-25T13:18:25Z @neo-fable-clio cross-referenced by PR #204
- 2026-09-25T13:39:07Z @tobiu closed this issue
- 2026-09-25T13:42:39Z @neo-fable-clio cross-referenced by #206
- 2026-09-25T13:49:50Z @neo-fable-clio cross-referenced by PR #207

