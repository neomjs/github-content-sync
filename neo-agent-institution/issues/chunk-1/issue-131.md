---
id: 131
title: 'Memories pane: the grid chrome paints through; the head has no gaps'
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T16:48:03Z'
updatedAt: '2026-09-12T20:39:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/131'
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
closedAt: '2026-09-12T20:39:18Z'
---
# Memories pane: the grid chrome paints through; the head has no gaps

## Context

Dogfooding screenshot of the Fleet Manager cockpit (2026-09-12, the Memories tab in the south register, dark skin, an agent with 56 sessions selected): the operator's read — "the design work was solid, but default list rules (many!) did not get reset; borders, selected items etc." and "the header there matters: 'What they remember' — no gaps, looks cramped". Both observations verified against the render and the sources below; the card typography, the meta line's content and the summary bodies read as designed.

## The Problem

The memories pane renders its session summaries and turns through engine grids (`AgentOS.view.fleet.memories.RowsGrid extends Neo.grid.Container`, the buffered row pool landed in #44). The pane's skin styles the CARDS (`.fm-memories-card`: a soft 1 px line, 6 px radius, the panel surface) and collapses the grid's header toolbar — but nothing resets the grid's own chrome, so the engine's data-table skin paints through the card design:

- every pooled cell carries the table lattice: `border-bottom` / `border-right` (and the first cell's `border-left`) in `--grid-container-border-color` — a second frame around every card, and the ruled gaps between cards that read as slabs;
- even rows take `--grid-container-cell-background-color-even` (purple-950 in the dark skin) — the alternating stripe behind the cards;
- the row under the pointer takes `--grid-cell-background-color-hover` (purple-900);
- the grid's default selection model marks the clicked row `.neo-selected` — the purple-600 band with contrast text that the screenshot shows behind "Fixing MCP Server Schema Validation Error" — although the surface has no selection semantics: the drill fires from the card click (`memories/Container.mjs:357`, `sessionDetailRequest`), never from a selection;
- the grid container's own 1 px border (`--grid-container-border-color`) frames the whole register.

The head is the second defect: `.fm-memories-head` declares `gap: var(--fm-space-2); align-items: baseline; flex-wrap: wrap` and nothing else — no space between the tab strip and the title, the authority words ("session summaries · query-time · not authority") and the meta line ride the title's baseline row, and the first card starts directly under the meta line. The skin's own header comment names why: the pane was authored for the pinned DRAWER, whose host "owns surface, padding, and rhythm — this root declares no frame". In the docked south register no host supplies that rhythm, so the head is cramped exactly there — and the §04 rhythm record in the same comment lists the off-rhythm survivors it deferred.

## The Architectural Reality

- Engine pin 7 (`node_modules/neo.mjs`, dev@28e56e1543): `resources/scss/src/grid/Body.scss` — cell background + `border-bottom`/`border-right` (lines 56–58), the first cell's `border-left` (71), the even-row background (50), the hover tint on `:not(.neo-selected):hover, .neo-hover` (115–119), the row-model selection colors on `.neo-selected` (104–107; the cell-model variant carries `!important`, 88–90); `resources/scss/src/grid/Container.scss:2` — the container border. The dark skin binds them in `resources/scss/theme-neo-dark/grid/*.scss`: border purple-800, even rows purple-950, hover purple-900, selected purple-600 + `--sem-color-text-neutral-contrast`.
- The engine exposes the colors as `--grid-*` custom properties (the full list is in the theme files) — a consumer can re-bind them per surface without touching the structural rules; the structural 1 px borders stay in layout even when transparent.
- `resources/scss/src/apps/agentos/fleet/memories/Container.scss`: the head rules (lines 11–15), the card vocabulary (lines 40–60), and the grid registers block (`.fm-memories-summary-grid, .fm-memories-turn-grid`, ~line 160) that collapses `.neo-grid-header-toolbar` to zero height at the skin layer — the precedent for "engine chrome reset here, per surface": the rules live in this file because the engine loads CSS per instantiated class name and a `memories/Grid.scss` would map to no class.
- `apps/agentos/view/fleet/memories/RowsGrid.mjs` declares no `selectionModel`; `grid.View` owns the engine default (the locked bodies carry `selectionModel: null` by design, `src/grid/Container.mjs:849`).
- `test/playwright/visual/FleetCockpitVisual.spec.mjs` has no memories-pane arm (the tasks pane has its 720 / 240 bands, #113) — no golden guards this surface today.

## The Fix

1. **Grid chrome reset in the registers block** of `memories/Container.scss` (the same block that collapses the header toolbar): the container border to 0; the pooled cells' `border` to 0 and their backgrounds (`--grid-container-cell-background-color`, `-even`) to transparent; the hover tint to transparent. Re-bind the `--grid-*` properties where the engine reads a variable; override the structural rule where it does not (the 1 px lattice). The card keeps its own frame and surface.
2. **The selection paint neutralized at the skin layer:** the engine instantiates a `RowModel` for a null `selectionModel` (`grid.Body#beforeSetSelectionModel` → `ClassSystemUtil.beforeSetInstance(null, RowModel)`), so a consumer cannot opt out at pin 7. The `--grid-rowmodel-selected-*` properties are re-bound on the registers (transparent surface, the card's ink), the mark's class stays, and neomjs/neo#18626 — the engine-side opt-out — is the retirement trigger for the neutralization.
3. **The pane root carries the panel rhythm** the roster root declares (`padding: var(--fm-space-4); gap: var(--fm-space-3)`): the head, the meta line, the registers and the actions rail sit on one rhythm in both live hosts — the docked south register and the vessel window — neither of which hands the pane a rhythm. The pinned-drawer host the skin's header cited no longer exists; that clause retires with it.
4. **A memories-pane visual golden, both skins**, in `FleetCockpitVisual.spec.mjs` at the register's band — red-first against today's render (the lattice, the stripe and the selection band are what the golden refuses).

## Acceptance Criteria

- [ ] On the summary and turn grids no engine grid chrome is visible: no cell or container borders, no even-row stripe, no hover tint, no selection band — asserted on computed styles at the cell level in the Neural Link witness over live cards (both registers, even rows included, the hovered cell, the marked row, the container frame) and pinned by goldens of both populated registers in both skins.
- [ ] The row the engine marks after a card click paints no selection band (its cell background stays transparent, asserted on computed styles), and the drill still fires `sessionDetailRequest` from the card click (existing coverage green). Reworded 2026-09-12 at implementation: the engine instantiates a `RowModel` for a null `selectionModel` (`grid.Body#beforeSetSelectionModel` → `ClassSystemUtil.beforeSetInstance(null, RowModel)`), so a consumer cannot opt out at pin 7 — the mark's class stays, its paint is neutralized at the skin layer; an engine-side opt-out (neomjs/neo#18626) is the retirement trigger for that neutralization.
- [ ] The pane carries the panel rhythm: root padding `--fm-space-4` and gap `--fm-space-3` (the head 16 px under the tab strip, the meta line one gap below, one gap before the first register), cited in the SCSS beside the rule and asserted on computed styles; the same root rule serves the vessel window.
- [ ] The header-toolbar collapse stays as it is (zero height, width kept — the engine measures flex columns through it).
- [ ] `FleetCockpitVisual.spec.mjs` gains a memories-pane arm in both skins (the empty pane: the rhythm); it is red before the fix and green after; the baseline stamp is re-stamped over the staged inputs. The populated registers' goldens (summary and turns, both skins) live in `FleetMemoriesNL.spec.mjs`, because only the wire populates them.

## Out of Scope

- The card typography, the meta line's content and the turn/drill vocabulary (solid as designed).
- The engine's grid theme itself: this is a consumer-side reset for a designed list. Retirement trigger: when the engine ships a headerless "designed list" grid variant (no lattice, no default selection), this pane adopts it and the reset block goes.
- #128 (roster card density) and #129 (cockpit chrome legibility) — their own lanes.

## Avoided Traps

- `display: none` on the header toolbar — the engine measures flex-sized columns through it; the collapse stays a height-zero skin rule.
- Expecting `selectionModel: null` to remove the model — at pin 7 it instantiates the default `RowModel`; the honest consumer move is the paint neutralization with the engine opt-out (neomjs/neo#18626) named as its retirement, never a claim that no selection exists.
- Fighting `!important` in the engine's cell-model selection rule — the memories registers run the row model, whose rule carries no `!important`; the re-bound properties suffice.
- A placement-keyed rhythm (docked vs. drawer) — the drawer host no longer exists; one root rule serves both live hosts.

## Related

#13 (parent: agentos design conformance — the live app must consume the token system it already loads) · #10 (the FM cockpit UI/UX epic; the sibling dogfooding tickets #128, #129) · #44 (the memories pane's browse and drill through buffered grids — where the header-toolbar collapse landed) · #113 (the tasks-pane visual bands, the golden precedent) · neomjs/neo#17265 / neomjs/neo#17268 (the §04 rhythm record and the pane-composition pass the skin's header comment defers to).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-12T16:41Z; no equivalent (#44 is closed and covers the grids' landing, not their chrome; #13 is the parent epic). A2A in-flight claim sweep: the last 30 messages (all read-states, 13:04Z–16:46Z) carry no `[lane-claim]` / `[lane-intent]` on the memories pane or the FM theming (the open claims are engine leaves: #18607, #18622, #18531). Memory Core rationale sweep: `query_raw_memories` on the symptom nouns at 16:44Z returned six unrelated session-initialization records (the semantic index reports embed-state-unavailable), so no prior decision on this surface is on record; the only decided history is #44 (the grids' landing and the header-toolbar collapse). Own-assignment sweep: my open same-surface tickets are #128 (roster cards) and #129 (cockpit chrome) — different surfaces; nothing on the memories pane.

Origin Session ID: 46156fd3-ef37-410c-a3fa-f843df795597

Retrieval Hint: "memories pane grid chrome reset selection model null head rhythm docked register"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46156fd3-ef37-410c-a3fa-f843df795597


## Timeline

- 2026-09-12T16:48:03Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-12T16:48:04Z @neo-fable-clio added the `bug` label
- 2026-09-12T16:48:04Z @neo-fable-clio added the `ai` label
- 2026-09-12T16:48:04Z @neo-fable-clio added the `design` label
- 2026-09-12T17:08:04Z @neo-fable-clio cross-referenced by #18626
- 2026-09-12T17:09:33Z @neo-fable-clio cross-referenced by PR #132
- 2026-09-12T17:26:59Z @neo-fable-clio referenced in commit `a8f176a` - "fix(fleet): the memories registers wear no grid chrome, and the pane carries the panel rhythm (#131)

The engine's data-table skin painted through the card design — the cell lattice, the even-row
stripe, the hover tint and the row-model selection band — and the head ran without gaps because
the pane was authored for a drawer host that owned the rhythm. The registers block resets the
chrome at the skin layer: the colors as the engine's own --grid-* properties re-bound per surface,
the structural lattice and cell padding overridden by a four-class rule (the engine's is three;
per-class CSS load order is not a contract). The pane root carries the roster root's panel
rhythm (padding --fm-space-4, gap --fm-space-3) in both hosts. The row selection stays: the
engine instantiates a RowModel for a null selectionModel, so its paint is neutralized and its
class is not (retirement trigger: an engine-side opt-out).

Coverage: FleetCockpitVisual gains the memories pane at the 720 band in both skins with the rhythm
and the register frame asserted on computed styles (red-first against the old CSS); FleetMemoriesNL
gains a chrome arm on the live cards (background, borders, padding; the engine-marked row paints
no band) — red on the first cut's two-class rule, which lost to the engine's lattice rule."
- 2026-09-12T17:46:45Z @neo-fable-clio referenced in commit `73d20f6` - "test(fleet): the populated memories registers are goldened in both skins (#131)

The first head's goldens showed only the empty pane and the witness measured one summary cell (review RA-1). The memories witness gains a second test: the scripted fleet serves fleetSessionMemories (two authored turns, one with a miniSummary headline), and the arm asserts every pooled cell of the summary register and, one drill down, of the turn register — bare background, border and padding, even rows included — plus the hovered cell painting no tint, the row the engine marks painting no band (a turn row too), and both container frames at zero; four goldens pin the paint in both skins through the real ViewportController#setTheme. The roster selection reveals the inspector on the rail, so the arm dismisses it with a mousedown outside the rail before the drill click and the captures; the goldens are the whole pane at a taller viewport so the cards are visible, not a sliver."
- 2026-09-12T20:39:18Z @tobiu referenced in commit `a44ce6a` - "Merge pull request #132 from neomjs/agent/131-memories-chrome-reset

fix(fleet): the memories registers wear no grid chrome, and the pane carries the panel rhythm (#131)"
- 2026-09-12T20:39:19Z @tobiu closed this issue
- 2026-09-13T11:50:29Z @neo-fable-clio cross-referenced by PR #18661
- 2026-09-13T19:10:55Z @neo-fable-clio cross-referenced by #136
- 2026-09-14T00:40:14Z @neo-fable-clio cross-referenced by PR #137

