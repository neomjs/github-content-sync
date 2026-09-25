---
id: 201
title: 'The cockpit bar grows above 1180 px: the state block stacks its pills'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T12:20:00Z'
updatedAt: '2026-09-25T13:18:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/201'
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
closedAt: '2026-09-25T13:18:31Z'
---
# The cockpit bar grows above 1180 px: the state block stacks its pills

## Context

Design pass on #13, leaf 3 (after #198, the frame gutter, and #200, the bar's control scale). Measured on `fix/199-cockpit-bar-scale` (= `dev@2291731` + #200's `f81afd8`) in the browser (Playwright, CSS px):

| viewport (container) | state block | bar height | buttons |
|---|---|---|---|
| 1400 (1350) | column: `fleet offline` over `wake off`, 41 px | **53** | 32 |
| 1280 (1230) | column, 41 px | **53** | 32 |
| 1181 … 1000 (1131 … 950) | row: 185 × 19 px | 44 | 32 |
| 800 … 760 (750 … 710) | row of two 14 × 16 marks | 44 | 32 |
| 600 (550) | marks; buttons at 5 px padding | 44 | 32 |

## The Problem

The `#23` collapse order declared three forms for the state block: wide stacks the two pills vertically — "the 50px band has the vertical space, so the two state axes stack" (`apps/agentos/design/institution-header-detail-ia.html:109-119`) — mid runs them in a row, narrow drops to marks. That vertical space was a side effect of the engine's 48 px buttons: a 60 px bar had 41 px of stack to give away. With the bar on the chrome scale (#199), the stacked pills are the bar's tallest child and set its height — a window wider than 1180 px gets a 9 px taller chrome bar than a narrower one, the one direction a collapse order must never run. Where the stack shows, the bar's spacer holds at least 418 px of empty width (at 1181 it spans 313–731); the row form needs 185.

Grace's note on #198 (review 5317380824): `--dock-edge-rail-size: 20px` (`cockpit/Container.scss:43`) is the very line `--fm-frame-gutter`'s comment defines the gutter by (`Viewport.scss:53-56`) — two declarations carry one value.

## The Architectural Reality

- `.fm-bar-state` (`resources/scss/src/apps/agentos/fleet/cockpit/Container.scss:112-119`) is `flex-direction: column` by default; `@container fm-cockpit (max-width: 1180px)` (`:284-290`) flips it to a row; the 760 query (`:292`) drops the words to marks. The bar is an engine toolbar (`apps/agentos/view/fleet/cockpit/Container.mjs:289-293`), which centers its items on the cross axis — a 19 px row sits centered on the 32 px buttons (measured: pills at top 62 on buttons at top 56).
- The state block is an engine container without a declared layout, so it renders the container default `{ntype: 'vbox', align: 'stretch'}` (`neo.mjs/src/container/Base.mjs:110`); the SCSS `flex-direction: column` was restating that default and the 1180 query overriding it. A container query reads the container's inline size, never a child's height: the bar cannot "stack only when the height is free"; each form is a declaration per width band.
- The pills' 19 px come from the chip family's tokens (`--fm-chip-pad-y`, `--fm-text-detail`; `fleet/mailbox/Chips.scss`); a two-row stack cannot fit inside the 32 px control line without breaking the family's one geometry.
- The order is stated three times: the SCSS record (`Container.scss:271-283`, with the 760 measurement), the composition comment (`Container.mjs:302-308`), the IA record (`institution-header-detail-ia.html:109-119,145`).
- `--fm-frame-gutter` is declared on the FM token block (`Viewport.scss:56`), an ancestor of every `.neo-dashboard` the cockpit projects; the bar already reads it (`Container.scss:97`). The engine reads `width: var(--dock-edge-rail-size)` with no fallback (`neo.mjs/resources/scss/src/dashboard/Container.scss:508,517,673-674`), so the var must resolve where the cockpit renders — it does, from the same scope.

## The Fix

1. The row form is the block's own layout, declared in the view: `layout: {ntype: 'hbox', align: 'center'}` on the state block in `Container.mjs`; the SCSS rule keeps only `gap: var(--fm-space-2)` and `min-width: 0`, and the 1180 container query goes (net negative lines). The collapse order has two forms: words in a row from 761 up, marks with titles at 760 and below. The bar is 44 px (32 + 2 × 6) at every width above 760. *(Corrected 2026-09-25 12:35Z: the first cut dropped the SCSS direction and fell back to the engine's vbox default — the row is a layout declaration, not an SCSS override.)*
2. The three statements of the order follow the code: the SCSS record (two forms, the 760 measurement kept), the composition comment in `Container.mjs`, the IA record's wide column (the row form; the "vertical space" sentence becomes the record of why the stack went).
3. `Container.scss:43` becomes `--dock-edge-rail-size: var(--fm-frame-gutter);` — the §04 proportion record stays the rationale for 20, its last paragraph naming the token as the value's home; `Viewport.scss:53-55` names the rail as a reader.
4. The visual goldens above 1180 move again (the cockpit shifts up 9 px: `cockpit-default-shell`, `cockpit-review-1280`, plus the pane captures that grow by the freed height); re-rendered from a full run alone with a by-eye read; the baseline stamp re-issued.

## Acceptance Criteria

- [ ] `.fm-cockpit-bar` measures 44 px at 1400, 1280, 1181 and 1000 wide, and `.fm-bar-state` is a 19 px row at each (probe as above); 760 and below keep their asserted marks form (`FleetCockpitVisual.spec.mjs:371`; `FleetCockpitBarCompositionNL` 800/520 unchanged).
- [ ] `cockpit/Container.scss` carries no `flex-direction` for `.fm-bar-state` and no 1180 container query; the block declares its hbox layout in `Container.mjs`; the SCSS record, the composition comment and the IA record state two forms.
- [ ] `--dock-edge-rail-size` reads `var(--fm-frame-gutter)`; the edge rail still measures 20 px in both themes (DOM probe of the engine's rail element).
- [ ] `npm run test-visual` alone: every moved golden updated in this leaf and named; the stamp re-issued; `check-visual-baselines` 0.

## Out of Scope

- The preset switch's noun and grouping (a segmented control) — the next leaf.
- The pills' own geometry and the chip family's tokens.
- The button family's colour, border and state conversions (Grace's 102 declarations) — later leaves.

## Avoided Traps

- Keeping the column form and shrinking the pills to fit 32 px: the roster and activity chips share the tokens; one geometry or none.
- A height-conditional stack: container queries cannot read a child's height, and a measurement in the view would move a layout decision out of the SCSS.
- Reversing the token direction (the gutter reading the rail): the rail var lives on the projected `.neo-dashboard` scope inside the cockpit, unreadable by the top toolbar.
- Writing the row into the SCSS rule (the first cut): it overrides the engine's layout class from the outside; the container's `layout` config is where a Neo view says how its items flow.

## Related

#13 (parent) · #197 / #198 (leaf 1: the frame gutter) · #199 / #200 (leaf 2: the bar's control scale — this leaf stacks on its goldens) · Grace's note on #198 (review 5317380824) · the design-pass read (issuecomment-5831663831)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 12:17Z; no equivalent (#199 is leaf 2). A2A claim sweep (last 60 min, all read-states): no claim on the bar's state block or the rail token. Memory Core: no prior decision beyond the `#23` record in the source. Own-assignment sweep: #199 is the only same-surface hit.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm cockpit bar state block column row 1180 bar height 53 44 dock-edge-rail-size fm-frame-gutter"


## Timeline

- 2026-09-25T12:20:00Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T12:20:01Z @neo-fable-clio added the `bug` label
- 2026-09-25T12:20:01Z @neo-fable-clio added the `ai` label
- 2026-09-25T12:20:01Z @neo-fable-clio added the `design` label
- 2026-09-25T12:20:31Z @neo-fable-clio added parent issue #13
- 2026-09-25T12:31:07Z @neo-fable-clio cross-referenced by PR #202
- 2026-09-25T12:35:27Z @neo-fable-clio referenced in commit `7ea5d49` - "fix(cockpit): the state pills keep their row at every width; the rail reads the gutter (#201)

Above 1180 px the bar was 53 px and below it 44: the state block's wide form stacked its two
19 px pills, the bar's tallest child once the buttons sat on the 32 px chrome scale. The row
form is the block's own hbox layout, declared in the view, instead of an SCSS override of the
engine's default vbox; the 1180 container query goes, the collapse order has two forms. The
three statements of the order follow the code. The dock edge rail's extent reads the frame
gutter token, so the 20 px the rails and both chrome rows share is one declaration."
- 2026-09-25T12:38:54Z @neo-fable-clio referenced in commit `1d8f961` - "fix(cockpit): the state pills keep their row at every width; the rail reads the gutter (#201)

Above 1180 px the bar was 53 px and below it 44: the state block's wide form stacked its two
19 px pills, the bar's tallest child once the buttons sat on the 32 px chrome scale. The row
form is the block's own hbox layout, declared in the view, instead of an SCSS override of the
engine's default vbox; the 1180 container query goes, the collapse order has two forms. The
three statements of the order follow the code. The dock edge rail's extent reads the frame
gutter token, so the 20 px the rails and both chrome rows share is one declaration."
- 2026-09-25T13:10:52Z @neo-fable-clio cross-referenced by #203
- 2026-09-25T13:18:32Z @tobiu closed this issue
- 2026-09-25T13:42:39Z @neo-fable-clio cross-referenced by #206

