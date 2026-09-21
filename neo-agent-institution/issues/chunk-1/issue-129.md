---
id: 129
title: 'Cockpit chrome legibility: the aggregate dot doubles the first swatch, hover equals pressed on presets, the vessel window is titled by the instance'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T13:17:49Z'
updatedAt: '2026-09-18T17:59:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/129'
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
closedAt: '2026-09-18T17:59:33Z'
---
# Cockpit chrome legibility: the aggregate dot doubles the first swatch, hover equals pressed on presets, the vessel window is titled by the instance

## Context

Three sightings from the 2026-09-12 dogfooding, each verified in source:

1. **Health bar** — the legend line opens with two green dots: the aggregate verdict dot (`health/Container.scss`, the `::before` on the bar, "one glanceable header signal ahead of the counts") sits flush against the first swatch's own dot (`● 0 working`). In the golden and live alike it reads as a doubled glyph, and when nominal both are green, so the verdict is indistinguishable from the count it precedes.
2. **Preset buttons** — `.neo-button.fm-preset-button:hover` and `.pressed` set the identical trio (`cockpit/Container.scss`: border-color `--fm-ink-dim`, background `--fm-panel`, color `--fm-ink`). A mouse resting on an inactive preset paints it as the active one; in the visitor's screenshot Overview (hovered) and Gereon-Demo (pressed) both looked pressed — measured `pressed` was true on one only.
3. **Vessel window title** — the popped-out inspector's OS window is titled `127.0.0.1:8093/fleet — Agent OS` (`VesselContainer.afterTearOutWindowConnect` → `pushInstanceTitle(windowId)`): the instance, not the pane. With the inspector and Memories both out, the two windows carry the same title and the window switcher cannot tell them apart.

## The Fix

1. Give the aggregate dot its own slot — separated from the legend by a divider or a wider gap, carrying the verdict only — or fold it into the FLEET tab's title dot; one signal, one place.
2. Pressed ≠ hover: the pressed preset takes the signal (`--fm-signal` border or underline, the way the active tab does); hover keeps the quiet lift.
3. The vessel window title names the pane first: `Agent detail · neo-fable-clio — <instance>`; `pushInstanceTitle` composes from the pane's title (and the selected record where the pane has one).

## Intake 2026-09-18

Drift probe non-empty (`VesselContainer.mjs`, `ViewportController.mjs` moved since filing), so all three premises were re-read at dev@412b451fcf — all hold: the aggregate `::before` dot (`health/Container.scss:17-31`), the identical `:hover` / `.pressed` trios (`cockpit/Container.scss:130-140`), and `pushInstanceTitle` composing `${label} — Agent OS` from the instance alone (`VesselContainer.mjs:522-532`).

**The third sighting is worse than filed.** `ViewportController#pushTearOutTitles` — the re-title sweep after an instance switch — iterates `cockpit.tearOutConnects`, a member that exists nowhere any more: not on the cockpit, not in the engine at pin 12 (the engine's own arm asserts the field "is no longer allocated on any Workspace"). The `?? {}` answers an empty map, so after a switch every torn-out window keeps the OLD instance's title. Same class as #161: a consumer reading a member its owner lost, hidden by a fail-closed default. The vessel truth lives in `nativeWindows.getOwner / getConnection(cockpitId, itemId)`; the sweep belongs on the cockpit (one title authority), over its declared panes.

Shapes chosen (design-led lane, captures in the PR for @tobiu's eyes):
1. The aggregate signal changes SHAPE, not only distance: a short vertical bar in the state colour with a wider gap — the cards' own left accent, so the verdict never reads as a swatch. Zero new hues.
2. Pressed takes the chrome's existing "selected" idiom, the signal underline the active dock tab carries; hover keeps the quiet lift (border `--fm-ink-dim`). Start fleet stays the only signal-bordered control on the bar.
3. `<pane title>[ · <selected resident>] — <instance>`, composed on the cockpit, pushed on connect and by a sweep that reads the Group's connections.

## Acceptance Criteria

- [ ] The health legend's first swatch shows one dot; the aggregate verdict is visually distinct; the golden is re-captured on purpose with the by-eye diff in the PR.
- [ ] A hovered inactive preset does not equal the pressed one (computed border-color differs); the pressed one is unique on the bar.
- [ ] Two vessels open at once carry distinct window titles; a unit arm on the title composition; the pop-out NL witness reads the vessel's `document.title`.
- [ ] After an instance switch every open vessel window carries the NEW instance in its title (the sweep reads the Group's connections; red on dev, where it iterates a member that no longer exists).

## Out of Scope

The card's own chrome (#123 and the density ticket).

## Related

#10 (parent epic); the density ticket filed alongside; #125 (the vessel layer).

Live latest-open sweep: the latest 20 open issues of neomjs/neo-agent-institution at 2026-09-12T13:14Z — none equivalent. A2A in-flight claim sweep: the last 30 messages carry no claim on this scope. Memory Core rationale sweep: nothing decided on these three. Own-assignment sweep: none on these surfaces.

Origin Session ID: fcdd7d71-e7bd-46e8-bd01-5d46ea620205
Retrieval Hint: "cockpit chrome aggregate dot preset pressed hover vessel title"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fcdd7d71-e7bd-46e8-bd01-5d46ea620205


## Timeline

- 2026-09-12T13:17:49Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-12T13:17:50Z @neo-fable-clio added the `bug` label
- 2026-09-12T13:17:50Z @neo-fable-clio added the `ai` label
- 2026-09-12T13:17:50Z @neo-fable-clio added the `design` label
- 2026-09-12T16:48:04Z @neo-fable-clio cross-referenced by #131
- 2026-09-12T17:09:33Z @neo-fable-clio cross-referenced by PR #132
- 2026-09-12T21:01:34Z @neo-fable-clio cross-referenced by #133
- 2026-09-13T19:10:55Z @neo-fable-clio cross-referenced by #136
- 2026-09-15T16:32:00Z @neo-opus-vega cross-referenced by #142
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T13:03:35Z @neo-fable-clio cross-referenced by #153
- 2026-09-18T13:40:14Z @neo-fable-clio cross-referenced by #155
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157
- 2026-09-18T15:21:07Z @neo-fable-clio cross-referenced by #160
- 2026-09-18T15:30:31Z @neo-fable-clio cross-referenced by #161
- 2026-09-18T16:58:52Z @neo-fable-clio cross-referenced by PR #167
- 2026-09-18T17:04:02Z @neo-fable-clio cross-referenced by #123
- 2026-09-18T17:31:00Z @neo-fable-clio cross-referenced by #168
- 2026-09-18T17:59:33Z @tobiu referenced in commit `d02fe83` - "Merge pull request #167 from neomjs/agent/129-cockpit-chrome-legibility

fix(cockpit): the verdict is a bar, pressed is not hover, a vessel window names its pane (#129)"
- 2026-09-18T17:59:33Z @tobiu closed this issue
- 2026-09-18T18:34:38Z @neo-fable-clio cross-referenced by #11

