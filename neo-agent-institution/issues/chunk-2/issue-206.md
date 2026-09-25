---
id: 206
title: 'The perspective switch reads as three buttons, not one choice'
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T13:42:38Z'
updatedAt: '2026-09-25T14:06:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/206'
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
closedAt: '2026-09-25T14:06:11Z'
---
# The perspective switch reads as three buttons, not one choice

## Context

Design pass on #13, leaf 5 — the "perspective switch noun" row of the part-1 read (issuecomment-5831663831). Measured on `dev@893ec3b` at 1400 wide (Playwright, CSS px): Overview / Focus / Review are three separate bordered buttons (79 / 60 / 66 × 32, `1px solid var(--fm-line)`, `var(--fm-panel-2)`) with the bar's 8 px gap between them; the pressed one carries `var(--fm-panel)` and a 2 px signal underline (`cockpit/Container.scss:142-171`). The switch was placed in the control bar as a first cut "operable-cold; the rail pane placeholder stays for the switcher leaf" (Grace, neo `#14616` / PR `#15004`, 2026-07-10, Memory Core) — this is that switcher leaf.

## The Problem

Three boxes with gaps between them read as three independent actions in the same row as Reconnect and Start fleet, which ARE independent actions. They are one choice with exactly one value: the engine's published `dock.perspective.active` presses exactly one of them (`CockpitPerspectives.buttons()`), and the visual suite pins that uniqueness (`FleetCockpitVisual.spec.mjs:302`). The pressed underline borrows the dock tab's idiom, so a reader sees tabs drawn as buttons, sitting directly above a real tab strip. A segmented control says what the thing is: one bordered field, three segments, one of them selected.

## The Architectural Reality

- The three buttons come from `apps/agentos/util/CockpitPerspectives.mjs` `buttons()` (fresh configs, `presetName`, `reference: fleet-preset-<name>`, `bind: {pressed}`); the cockpit spreads them into the toolbar's items (`view/fleet/cockpit/Container.mjs:293`). Specs pin the button class and references (`.fm-preset-button`, `fleet-preset-*`, `presetName`) in unit, visual and e2e suites — they stay.
- The engine ships no segmented or grouped button control (`src/button/`: Base, Effect, Menu, Split; `src/toolbar/`: ActionButton, Base, Breadcrumb, Paging), so the group is a plain container with an hbox layout around the three declared buttons — layout declared in the view, geometry in SCSS, like the state block (#201).
- `Container.mjs` sits at 999 of 1000 lines (`checkAppFileSizes.mjs`): the group factory lives in `CockpitPerspectives` (`group()` returning the container config with `buttons()` as its items), and the view's spread becomes one call — net zero lines in the view.
- The hover-vs-pressed contract (`FleetCockpitVisual.spec.mjs:288-324`) reads the border colour as the hover lift; segments have no border of their own, so the lift moves to the background (a half step from `--fm-panel-2` toward `--fm-panel`) and the spec reads `backgroundColor` instead. The pressed marker stays the chrome's underline (selected ≠ hovered by the underline AND by the background).
- The quiet base rule is shared with Reconnect (`:142-171`); the group takes the border, radius and background, the segments drop theirs and gain a 1 px divider (`border-left` on every segment but the first). The 760 mark regime never drops the view labels; at the 570 wrap the group wraps as one unit.

## The Fix

1. `CockpitPerspectives.group()` → `{ntype: 'container', cls: ['fm-preset-group'], role: 'group', ariaLabel: 'Perspective', layout: {ntype: 'hbox', align: 'center'}, items: this.buttons()}`; `Container.mjs:293` spreads no more, it places the group. JSDoc on both.
2. `cockpit/Container.scss`: `.fm-preset-group` carries `border: 1px solid var(--fm-line)`, `border-radius: 6px`, `background: var(--fm-panel-2)`, `overflow: hidden`, height on the bar's control line; `.neo-button.fm-preset-button` inside it: `border: 0`, `border-radius: 0`, transparent background, `& + &` divider, hover = the half-step background + ink, pressed = `var(--fm-panel)` + ink + the underline. Reconnect keeps the quiet base as is.
3. `FleetCockpitVisual.spec.mjs:296-323`: the paint reads `backgroundColor`; assertions: hover lifts the background and settles, hovered background ≠ selected background, hovered shadow none, selected shadow ≠ none.
4. Goldens with the bar re-render from a full run alone (`cockpit-default-shell`, `cockpit-review-1280`, `cockpit-vessel-314`; the darwin e2e bar-composition captures at 800/520 on the next NL battery run); stamp re-issued.
5. Rider, pixel-neutral (Grace's #204 polish, re-stamp only): the recorded-exception markers for `Viewport.scss` (the agent-button glyph and the welcome eyebrow) move onto the literal's line so the #203 census grep lists them; the card monogram's `font-weight: 600` after the display role goes (the role carries it).

## Acceptance Criteria

- [ ] At 1400, 1280, 800 and 600 wide the three presets sit inside one `.fm-preset-group` box (one border, 6 px radius); the buttons measure no border of their own; dividers between segments (probe: rects + computed borders).
- [ ] Exactly one segment is pressed at any time (existing assertion); hover and pressed differ by background and by the underline (the updated computed-style test).
- [ ] At 560 wide the group wraps as one unit onto the bar's second row (the vessel-narrow rule), nothing clips mid-word.
- [ ] `Container.mjs` stays at or under 1000 lines; unit, visual and e2e specs that name `.fm-preset-button` / `fleet-preset-*` pass unchanged.
- [ ] `npm run test-visual` alone: every moved golden updated and named; the stamp re-issued; `check-visual-baselines` 0.

## Out of Scope

- A generic segmented-control component in the engine (a token/component decision for the engine, not this leaf).
- Radio semantics (`role="radio"`, `aria-checked`) on the engine's button — the engine's `pressed` carries its own ARIA; the group is labelled.
- The button family's colour and border conversions (Grace's 102), the top toolbar's scale.

## Avoided Traps

- Drawing the switch as tabs (underline only, no box): it would sit directly above the dock's real tab strip and read as a second strip of panes.
- Keeping three boxes and only tightening the gap: still three actions.
- Adding the wrapper in `Container.mjs`: the file is one line under the size gate.

## Related

#13 (parent) · #199 / #200 · #201 / #202 · #203 / #204 (the same bar, in order) · the part-1 read (issuecomment-5831663831) · neo `#14616` / PR `#15004` (the switch's first cut, "the switcher leaf" deferred)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 13:41Z; no equivalent. A2A claim sweep (last 60 min, all read-states, 13:40Z): no claim on the preset switch. Memory Core: Grace's 2026-07-10 record of the preset bar as a first cut awaiting the switcher leaf. Own-assignment sweep: no open same-surface hit (#203 merged 13:39Z).

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm cockpit preset switch segmented control fm-preset-group one choice underline hover background"

## Timeline

- 2026-09-25T13:42:38Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T13:42:40Z @neo-fable-clio added the `enhancement` label
- 2026-09-25T13:42:40Z @neo-fable-clio added the `ai` label
- 2026-09-25T13:42:40Z @neo-fable-clio added the `design` label
- 2026-09-25T13:42:52Z @neo-fable-clio added parent issue #13
- 2026-09-25T13:49:50Z @neo-fable-clio cross-referenced by PR #207
- 2026-09-25T13:56:26Z @neo-fable-clio cross-referenced by #208
- 2026-09-25T14:06:11Z @tobiu closed this issue

