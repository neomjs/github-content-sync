---
id: 1033
title: Home's canvas replays the team's record on the engine's round table
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-20T15:59:00Z'
updatedAt: '2026-11-20T15:59:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1033'
author: neo-fable-mnemosyne
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
# Home's canvas replays the team's record on the engine's round table

unowned-rationale: Vega, who built Home's field, asked for this leaf to be filed unowned: she takes it after the Engine 14 cut (Mon 23 Nov) unless a seat with room is faster. No curated label: one design question is open (The Fix, step 1).

## Context

On 20 Nov 2026 the operator looked at Home's canvas while the Portal's hero was being rebuilt: it looks like a spider web with hover animations, not a simulation, and it is below the current bar (his read, recorded in session `9ff53ea4-2f77-4c2f-b7fe-0fdeff46a37f`). The same day the engine got a scene made for this: `Neo.canvas.RoundTable` with its host `Neo.app.roundtable.Canvas` (neomjs/neo#21568, merged with neomjs/neo#21581 at `02adfb3bc8`). The Portal's hero is its first consumer. Vega agreed by A2A that Home should be the second: "One engine scene with two consumers beats an app-local copy that receives no engine fix."

## The Problem

`AgentOS.canvas.Home` draws motes that link where they pass, and one mark per rostered agent on a ring. It shows that agents exist and which are up. It does not show what the product's own line promises on the same screen: agents "that review each other's work", and "You decide what merges."

## The Architectural Reality

- `apps/agentos/canvas/Home.mjs` (940 lines) is the renderer and `apps/agentos/view/home/Canvas.mjs` its host (`Neo.app.SharedCanvas`), mounted by `apps/agentos/view/home/Container.mjs`. The field is handed the roster read's `{total, up}`.
- The engine scene is fed a record, not a count: `{from, to, caption, unit, groups, seats, items}`, an item being `[number, title, seat, login, opened, merged, reviews]` with times in minutes from `from` (class docs of `src/canvas/RoundTable.mjs`). The host takes `record` or `recordUrl`, a `look` (`palettes` per theme, optional `measures` and `pace`), and the statement to draw around: `blockIds` (nothing is drawn under them) and `bandIds` (no seat stands in their rows). It reads stillness from `--motion-base` itself.
- The engine scene took three solved parts from this field: the quiet rects under the hero's lines (#365), the still frame and the pixel-ratio buffer.
- The specs read the renderer: `test/playwright/e2e/agentos/HomeField.spec.mjs` and `HomeMergeQueueDoorNL.spec.mjs`, the unit specs `test/playwright/unit/apps/agentos/canvas/home.spec.mjs` and `view/homeCanvas.spec.mjs`, and `test/playwright/visual/homeState.driver.mjs`. The engine host answers through `readStats()`: mode, seats with their boxes, the strip's box, items, landed, frames, still, theme, size.
- The engine arrives through `"neo.mjs": "github:neomjs/neo#dev"`; the lockfile pins the commit.

Design authority: "Home gets a live canvas, at least at the portal hero's bar" (#244, the field's ticket). The Portal hero's bar is neomjs/neo#21568 since 20 Nov 2026.

## The Fix

1. **Decide what Home replays.** The Portal replays a week of a public repository, written by a build script. Home runs inside an installed app on one team's plane. The builder settles the record's source with the home's steward: the team's own merged work as the app already reads it, or a window the plane serves. A record written at build time would show another team's week.
2. Move the lockfile's engine pin to a commit that holds neomjs/neo#21581.
3. `apps/agentos/view/home/Canvas.mjs` extends `Neo.app.roundtable.Canvas` and names the app's palettes (the cockpit's ink, `fmPalette.mjs`) and the statement's components.
4. `apps/agentos/canvas/Home.mjs` and what only it uses are deleted.
5. The specs port to `readStats()`: the arms that read motes and marks give way to arms that read seats, the strip and stillness, and the visual driver's goldens are retaken.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.view.home.Canvas` | this ticket | extends `Neo.app.roundtable.Canvas`, keeps `ntype: 'fm-home-canvas'` | none | class docs | unit spec |
| `AgentOS.canvas.Home` | this ticket | removed | none | none | grep |
| Home without a record | `Neo.canvas.RoundTable`: "while it has no record, the scene draws its ground alone" | the ground alone, no error | none | none | unit arm |

## Acceptance Criteria

- [ ] AC-1: Home's canvas is `Neo.canvas.RoundTable` through its engine host; `apps/agentos/canvas/Home.mjs` is gone and nothing imports it.
- [ ] AC-2: the record Home replays is the installed team's own, and the PR names its source with the steward's agreement.
- [ ] AC-3: nothing of the scene stands under the hero's lines or its two actions, at the default window size and at a narrow one (the engine's e2e check `touching` in `HomeHeroRoundTableNL.spec.mjs`, ported).
- [ ] AC-4: reduced motion shows the end state and draws no further frame.
- [ ] AC-5: without a plane, and before the first read answers, Home shows its ground and its lines without a console error.
- [ ] AC-6: the PR shows Home whole, as released and as changed, in both themes, with a cold look.
- [ ] AC-7 (post-merge): the installed Fleet Manager shows it against the team's plane.

## Out of Scope

- A change to the engine scene. What Home needs and the scene lacks is an engine ticket.
- Home's lines, its two actions and the operator's line.
- The Portal's hero.

## Avoided Traps

- **A second renderer that copies the round table into the app.** That is the state this leaf ends.
- **Feeding it the Portal's file.** That file is one public repository's week, dated at a release.
- **`ctx.filter` for the feather.** The engine blurs with a shadow because Safari has no canvas filter.

## Related

neomjs/neo#21568 · neomjs/neo#21581 · #244 (the field) · #365, #366, #875 (its fixes) · #505 (the view inventory's `home` key; that epic holds 25 leaves, so this one is not linked under it)

Decision Record impact: none known; the builder checks this repository's decisions for Home at intake.

Live latest-open + closed sweep: the latest 20 open issues, and open and closed searches (`home canvas`, `round table`, `home field`, `RoundTable`; Out of Scope read on #244) at 2026-11-20T15:56Z; no equivalent. 59 open issues, the red band; no open ticket owns Home's canvas, so this is filed. A2A sweep (all read states, the last 60 minutes): no claim on Home. Memory Core sweep ("Institution home canvas field round table"): no prior decision. Own-assignment sweep: #9, #391, #534, #727, #823, none on this surface. Structure map: not run; no `ai/` file and no new placement.

Origin Session ID: daf6414f-ebec-4af1-a4ea-631c110ac661
Retrieval Hint: `query_raw_memories("Institution home canvas round table second consumer record")`

## Timeline

- 2026-11-20T15:59:01Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-20T15:59:02Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-20T15:59:02Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-20T15:59:02Z @neo-fable-mnemosyne added the `design` label

