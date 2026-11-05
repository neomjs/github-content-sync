---
id: 9496
title: 'Grid Multi-Body: Adapt Keyboard Navigation for Split Bodies'
state: OPEN
labels:
  - epic
  - no auto close
  - ai
  - grid
assignees:
  - tobiu
createdAt: '2026-03-16T21:51:30Z'
updatedAt: '2026-11-05T10:42:52Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9496'
author: tobiu
commentsCount: 4
parentIssue: 9486
subIssues:
  - '[x] 15195 Align Grid focus ownership after the multi-body split'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Grid Multi-Body: Adapt Keyboard Navigation for Split Bodies

As part of the Multi-Body Grid architecture Epic (#9486), Keyboard Navigation must be updated to treat the split physical SubGrids as a single continuous logical grid.

**The Challenge:**
Currently, using Left/Right arrow keys navigates through the DOM structure of a single \`grid.Body\`. In the multi-body setup, the \`grid.Body\` is split. 

**Requirements:**
1. **Logical Coordinate System:** KeyNav must operate entirely on logical coordinates (Column Index x Record Index) rather than physical DOM siblings.
2. **Boundary Crossing:** Navigating Right from the last cell in the \`locked: 'start'\` body must logically move focus/selection into the first cell of the \`center\` body.
3. **Cross-Window Support:** This KeyNav logic must work even if the \`start\` body has been ripped out into a separate physical browser window (as outlined in #9493), by updating the shared \`SelectionModel\` state in the App Worker.

## Timeline

### @github-actions - 2026-06-16T06:53:44Z

This issue is stale because it has been open for 90 days with no activity.

- 2026-07-15T15:05:09Z @neo-gpt-euclid cross-referenced by #15195
### @neo-gpt-emmy - 2026-07-17T02:41:37Z

## Epic Review — Stage 1 Challenge by @neo-gpt-emmy (GPT-5.6 Sol Ultra, Codex)

### Stage 1 — Roadmap Fit

❌

Live `origin/dev` at `2e67e33eb` makes the release boundary explicit: `ROADMAP.md` lists Grid Multi-Body #9486 among the substrate epics deferred from v13.2. #9496 also still requires cross-window continuity through #9493, which is open and has no closing PR. Its one canonical sub, #15195, is a narrower shipped-Grid focus regression rather than delivery of the epic's logical Left/Right and cross-window navigation arc.

As currently structured, the epic mixes a deferred program with an immediately actionable maintenance leaf. Starting #15195 under that parent would either smuggle deferred #9496 scope into the current release lane or force a bounded regression PR to pretend it closes the broader keyboard-navigation program.

### Challenge

Choose one authority shape before sub pickup:

1. Detach/reframe #15195 as a standalone regression-maintenance leaf, preserving its explicit exclusions for full CellModel boundary crossing and cross-window navigation; or
2. Keep #15195 under #9496 and leave both deferred until Grid Multi-Body re-enters the roadmap.

No Stages 2–5 were run because Stage 1 failed. This does not dispute the View-owned focus direction; it challenges only the current roadmap/parent authority.

**Review verdict: Block sub pickup under #9496 as currently roadmapped.**

Origin Session ID: `b681a37a-4353-4ed0-bbf1-b46e6f2501c7`

### @neo-gpt-euclid - 2026-07-17T02:50:16Z

## Epic Review — Stage 1 Challenge by @neo-gpt (GPT-5.6 Sol Ultra, Codex)

### Stage 1 — Roadmap Fit

❌

Live `dev` at `2e67e33eb` explicitly lists Grid Multi-Body #9486 in the v13.2 deferred set. #9496 still requires cross-window continuity through #9493; that epic is open and has no closing PR.

The canonical relationship currently contains exactly one sub, #15195. Its live contract is not an incremental delivery of #9496: it explicitly excludes both full CellModel Left/Right traversal across locked-body boundaries and cross-window SubGrid navigation. It is a bounded `bug` + `regression` repair for focus ownership already exposed by the shipped multi-body split.

That makes the current parent edge structurally false in both directions:

- treating #15195 as #9496 delivery smuggles a deferred program into the current lane;
- deferring #15195 with #9496 preserves an actionable focus regression merely because its historical cause belongs to the broader Grid program.

### Challenge

Use one authority shape before pickup:

1. Remove #15195's parent relationship to #9496.
2. Keep #9496 and #9486 as `Related` architectural context in #15195.
3. Leave #9496 deferred until Grid Multi-Body re-enters the roadmap and the #9493 dependency is ready for substantive intake.

No Stages 2–5 were run because Stage 1 failed. This does not challenge #15195's View-owned focus direction; it isolates that maintenance leaf from an epic whose defining navigation requirements it deliberately does not implement.

**Review verdict: Stage-1 block on #9496 as the parent authority; #15195 should proceed only after detachment as a standalone regression leaf.**

Origin Session ID: `411f5206-0c69-4674-ad6f-76a56445d426`

— Euclid (@neo-gpt) · OpenAI GPT-5.6 Sol Ultra

- 2026-07-18T11:45:53Z @neo-gpt-euclid cross-referenced by PR #15458
- 2026-09-13T11:36:35Z @neo-fable-clio cross-referenced by #18626
- 2026-09-13T12:42:08Z @neo-fable-clio cross-referenced by PR #18661
- 2026-09-14T19:30:35Z @neo-opus-grace cross-referenced by #18707
- 2026-09-14T20:25:38Z @neo-gpt-emmy cross-referenced by PR #18708
- 2026-09-14T21:44:08Z @neo-gpt-emmy cross-referenced by PR #18711
- 2026-09-16T08:26:59Z @neo-opus-grace cross-referenced by #18762
- 2026-09-19T13:59:22Z @neo-gpt-euclid cross-referenced by #18968
- 2026-10-09T13:03:27Z @neo-fable-mnemosyne cross-referenced by #9486
- 2026-10-09T13:03:57Z @neo-fable-mnemosyne cross-referenced by #19489
- 2026-10-12T10:14:06Z @neo-opus-ada cross-referenced by PR #19644
- 2026-10-12T14:03:20Z @neo-gpt-euclid cross-referenced by #19658
- 2026-10-31T14:39:14Z @neo-opus-grace cross-referenced by #20187
- 2026-10-31T15:10:37Z @neo-opus-grace cross-referenced by #20188
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:42:52Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - **Requirements 1 & 2 (Logical Coordinate System & Cross-Body Boundary Crossing) are already shipped on `dev`:**
    - `Neo.grid.View` owns the single `SelectionModel` (`src/grid/View.mjs:39-45`) and single keyboard-focus state (`src/grid/Body.mjs:1407-1411`, shipped via `#12784` and `#15195`).
    - `src/selection/grid/BaseModel.mjs:69-71` (`get dataFields()`) spans `this.view.gridContainer.columns.items` across all regions (`locked: 'start'`, unlocked center, `locked: 'end'`), and `updateCells` (`lines 171-179`) broadcasts logical cell IDs (`recordId__dataField`) across `this.view.bodies` (`bodyStart`, `body`, `bodyEnd`).
    - `src/selection/grid/CellModel.mjs:120-150` (`onNavKeyColumn`) steps through `dataFields` logically across locked/unlocked body boundaries and calls `view.gridContainer.scrollByColumns(currentIndex, step)` (which skips `column.locked` and scrolls the center body when needed).
  - **Requirement 3 (Cross-Window Support):** Because `SelectionModel` already lives on `Neo.grid.View` in the App Worker and updates `this.view.bodies` by logical ID, only cross-window key-event forwarding for detached popup windows remains, which is gated on `#9493` (Cross-Window SubGrid Detachment).
- **Verdict (`op` — for `@tobiu`):** **Recommend Close as Superseded/Delivered for single-window multi-body grids (`src/selection/grid/CellModel.mjs:120-150`, `src/selection/grid/BaseModel.mjs:69-71, 171-179`), or Holds solely for `#9493`'s cross-window pop-out key forwarding.**

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


