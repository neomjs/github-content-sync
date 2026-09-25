---
id: 213
title: 'The Golden Path as a graph: a canvas-worker pane draws the computed route''s spine and its citations'
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T16:56:53Z'
updatedAt: '2026-09-25T18:57:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/213'
author: neo-fable-clio
commentsCount: 1
parentIssue: 9
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T18:57:46Z'
---
# The Golden Path as a graph: a canvas-worker pane draws the computed route's spine and its citations

## Context

The operator's refocus of 2026-09-25 names "GP and graph inside FM" as the high-ROI work. Two of the three pieces exist as lanes: the Brain's one wire read of the computed Golden Path (neo-agent-brain #496 / PR #499, `fleetGoldenPath`: the `computed-route.v1` sidecar passed through with its freshness, the corpus-projection admission and the REM counts) and the text pane that renders it (#210, Grace). The graph is the third piece, and #210 puts it out of its own scope on purpose. It needs nothing new on the wire: the envelope already carries a graph — ranked items with scores and citations, and citations shared across items are its edges.

## The Problem

The Golden Path is the picture the institution steers by. As text it is a list; as a graph it shows why the route runs the way it does: which items lean on the same evidence, how far apart the top of the route is from its tail, and what the route's freshness means for the whole picture. The cockpit has no graph surface at all, and the engine's canvas worker — one per window group since #19103, the 13.2 slice the release story leads with — has no showing in the product that ships on it.

## The Architectural Reality

- **Data:** `fleetGoldenPath` (#499) returns `{capability, admission, rem, route}`; `route` is the sidecar as written — `status` (fresh / empty / degraded …), `kind` (`computed-ranked` | `current-focus-substitution` | `none`), `freshness {status, checkedAt, expiresAt}`, `provenance {producer, runId, algorithmVersion, citations}`, `items[{id, title, score, rank, citations[]}]` (`computedRouteResult.mjs:117-135`). The pane synthesizes nothing: rank order and scores are the producer's; the only derived facts are positions.
- **Rendering primitive:** `Neo.component.Canvas` (app side: an offscreen `<canvas>` with `monitorSize`, a theme map, `getCanvasId`) hands the surface to a `Neo.canvas.Base` renderer on the canvas worker (`initGraph({canvasId, windowId})`, `render()`, `updateSize`, mouse handlers, `setTheme`) — the shape `Neo.canvas.Sparkline` and the portal's hero canvas (`apps/portal/view/home/parts/hero/Canvas.mjs` over `Neo.app.SharedCanvas`) already use. The AgentOS app must switch the canvas worker on in its `neo-config.json` (dev and the packaged stage, `harness/pack.mjs` copies it).
- **Layout is pure:** a module computes nodes and edges from the envelope — the route as a ranked spine (rank order along one axis, node weight from score), each item's citations as satellites, a citation shared by several items drawn once with an edge to each — deterministic for a given envelope, so it is unit-tested without a canvas and the renderer only draws.
- **States:** the same four as #210 (current / last-known-good / withheld / unavailable), derived the same way; the currency line sits above the canvas as text (shared derivation with #210 where it lands first), the graph's tone follows it (a withheld or expired route is drawn dim, never as current).
- **Placement:** a south-strip pane beside the catch-up and Golden Path text panes (`view/fleet/catchup/Container.mjs` is the reading-surface precedent; the cockpit registers panes in `view/fleet/cockpit/Container.mjs:150-200`); a pane that can be popped out like the others.
- **Design gates:** the token system (#13) for every colour and type on the canvas (the renderer reads the resolved theme through the canvas's theme map); the visual harness (#11) for goldens in both themes.

## The Fix

1. `apps/agentos/util/GoldenPathGraphLayout.mjs` (pure, `Neo.core.Base` statics): `layout(envelope, {width, height})` → `{nodes: [{id, kind: 'item'|'citation', x, y, r, label, rank, score}], edges: [{from, to}], tone}`; unit spec over #499's fixture envelopes (fresh two-item route with shared citations, empty, expired, withheld, unavailable).
2. `apps/agentos/canvas/GoldenPathGraph.mjs` (`Neo.canvas.Base`): draws nodes, edges and labels from a layout in the theme's ink/signal/line tokens; hover names the item (title, score, rank); `setTheme` re-tones.
3. `apps/agentos/view/fleet/goldenpath/GraphPane.mjs`: the pane — a currency header + a `Neo.component.Canvas` whose registration hands the layout to the renderer on resize and on every envelope change; bound to the same provider leaf the text pane reads; registered in the south strip; `useCanvasWorker: true` in `neo-config.json` (dev + stage).
4. Goldens for the four states in both themes through #11; the baseline stamp re-issued.

## Acceptance Criteria

- [ ] AC-1 The layout module produces the same nodes and edges for a fixture envelope on every run (unit spec per state); a citation shared by two items appears once with two edges; `kind: 'none'` yields no nodes and the state's tone.
- [ ] AC-2 The pane renders on the canvas worker (the renderer registers per `canvasId`; the visual harness captures the four states in both themes).
- [ ] AC-3 Hovering an item node names it (title · rank · score) through the canvas mouse path; nothing else is interactive in v1.
- [ ] AC-4 The pane synthesizes nothing: no re-ranking, no merging, no cached route — reviewed against the diff.
- [ ] AC-5 Against the live plane the graph shows the day's real state (today: withheld, drawn dim with the last route's `capturedAt`), never a route presented as current.

## Out of Scope

- The Brain read (#499) and the text pane (#210).
- Concept or edge graphs of the Memory Core (Institution #8's COP).
- Force-directed physics, zoom or drag — v1's layout is deterministic and fits the pane.
- Ranking changes to the Golden Path (neo-agent-brain #122's leaves).

## Avoided Traps

- Drawing in the App Worker or the main thread with DOM nodes: the canvas worker is the point, both for the frame budget and for the story.
- A layout computed in the renderer: it would be untestable without a canvas and would drift from the pane's state derivation.
- A second wire read for "the graph": the envelope's citations are the edges; a graph verb would duplicate #499.

## Related

#210 (the text pane, Grace) · #9 (parent) · #13 (design gates) · #11 (goldens) · neo-agent-brain #496 / #499 (the read) · neo #19103 (the canvas worker per window group)

Live latest-open sweep: the latest 20 open issues checked at 2026-09-25 16:55Z; no equivalent (#210 is the text pane and excludes a graph). A2A claim sweep (last 60 min, all read-states): Grace's #210 claim names the graph as out of scope; no other claim. Memory Core: semantic recall unavailable today (embed drain). Convergence artifact for the lead-role fan-out guard: #210 + #499 fix the data contract; the operator's directive names the graph.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "golden path graph pane canvas worker route spine citations layout fleetGoldenPath"

## Timeline

- 2026-09-25T16:56:53Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T16:56:55Z @neo-fable-clio added the `enhancement` label
- 2026-09-25T16:56:55Z @neo-fable-clio added the `ai` label
- 2026-09-25T16:56:55Z @neo-fable-clio added the `design` label
- 2026-09-25T16:57:11Z @neo-fable-clio added parent issue #9
### @neo-fable-clio - 2026-09-25T17:00:05Z

Progress 2026-09-25 17:05Z: Fix step 1 is on the branch `feat/213-golden-path-graph` at `8566ad5` — `apps/agentos/util/GoldenPathGraphLayout.mjs` (the pure layout: spine by rank, weight by score, one node per distinct citation placed between its citing items, an edge per citing item; the four states from the envelope's verdicts, fail-closed) with `goldenPathGraphLayout.spec.mjs`, 11 arms green, the full unit suite green. Next: the `Neo.canvas.Base` renderer, the pane with its `Neo.component.Canvas`, `useCanvasWorker` in the app config, goldens for the four states. The PR opens with the goldens.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0fbfde3a-e817-4859-9351-2269eabdda9a


- 2026-09-25T17:00:35Z @neo-fable-clio referenced in commit `811b93c` - "feat(goldenpath): the graph pane's layout — the route as a ranked spine, its citations as the graph (#213)

A pure util turns the fleetGoldenPath envelope into nodes and edges a canvas-worker renderer
only draws: items on a spine in rank order with the producer's score as their weight, one node
per distinct citation placed between the items that cite it, an edge per citing item. The pane's
state is read from the envelope's verdicts, fail-closed. Deterministic, unit-tested per state."
- 2026-09-25T17:48:05Z @neo-opus-grace referenced in commit `0466ecd` - "feat(cockpit): a Golden Path pane renders the computed route as text with its currency (#210)

The south strip gains a Golden Path tab. It renders the fleetGoldenPath envelope in a fixed order:
- first, a currency line in the producer's words;
- then the route's items, in the producer's order, from a pane-local Store;
- last, the producer's provenance.

The pane synthesizes, ranks and caches nothing.

A route reads as current only when three things hold: the admission admits it, the route is fresh,
and it has not expired. Otherwise it is withheld and shown as the last known good route, with the
producer's reason. A degraded source (route file missing or invalid) and an unwired or failed read
say so instead of showing an empty route.

The envelope is cockpit provider truth, because more than one pane reads it (the graph pane of #213
binds the same leaf). The read lands it in the goldenPathEnvelope leaf through GoldenPathEnvelope,
which also holds the currency both panes derive. The landing is closed:
- every declared key is written on every read, so a block the wire sends as null lands as its blank;
- the reason is that setData drills objects into leaf paths, and an ancestor rebuild stops at a null
  block, while an omitted key keeps its old value;
- a real-provider arm pins this, with the raw wire envelopes as its control.

A bound envelope can land while the pane is still constructing, so the pane applies an envelope once
its Store exists rather than once construction completes.

The cockpit Controller (998 lines) and Container (999) sat at the 1,000-line bar, so two
responsibilities move out whole:
- the catch-up and Golden Path owners become a ReadingSurfacesController layer between the liveness
  layer and the intent layer;
- the perspective share beat's logic moves into CockpitPerspectives, with thin wrappers left
  behind.

The Neural Link spec lands the pane's four states through the cockpit's own write and captures them
in both skins. The three cockpit baselines that carry the stream tab strip are re-rendered: the new
tab sits under the diff threshold, so they passed without it."
- 2026-09-25T17:48:08Z @neo-opus-grace cross-referenced by PR #215
- 2026-09-25T17:56:58Z @neo-fable-clio referenced in commit `2a8f4a3` - "feat(cockpit): the Golden Path graph pane draws the route on the canvas worker (#213)

A south-strip pane binds the cockpit provider's goldenPathEnvelope leaf and hands the
envelope to a canvas-worker renderer: the computed route as a ranked spine, its citations
as satellites, a citation shared by several items placed once with an edge to each, the
currency line above. The layout is a pure module unit-tested without a canvas; the
renderer only draws; the state is the envelope's own verdicts, fail-closed. Hovering an
item names it through the engine's canvas mouse path. Goldens for the four states in both
skins, plus the hover; the shipped dock document and the view-topology table follow."
- 2026-09-25T17:58:01Z @neo-fable-clio cross-referenced by PR #216
- 2026-09-25T18:26:29Z @neo-fable-clio referenced in commit `753c95d` - "feat(cockpit): the Route graph pane draws the Golden Path on the canvas worker (#213)

A south-strip pane binds the cockpit provider's goldenPathEnvelope leaf and hands the landed
envelope to a canvas-worker renderer: the computed route as a ranked spine, its citations as
satellites, a citation shared by several items placed once with an edge to each, the currency
line above. The layout is a pure module unit-tested without a canvas; the renderer only draws;
what is drawn follows the cockpit's one currency reading (GoldenPathEnvelope): current in the
signal, withheld dim as the last known good route, the rest nothing. Hovering an item names it
through the engine's canvas mouse path. Goldens for the currency words in both skins plus the
hover; the shipped dock document and the view-topology table follow."
- 2026-09-25T18:30:22Z @neo-fable-clio referenced in commit `2a0c28d` - "feat(cockpit): the Route graph pane draws the Golden Path on the canvas worker (#213)

A south-strip pane binds the cockpit provider's goldenPathEnvelope leaf and hands the landed
envelope to a canvas-worker renderer: the computed route as a ranked spine, its citations as
satellites, a citation shared by several items placed once with an edge to each, the currency
line above. The layout is a pure module unit-tested without a canvas; the renderer only draws;
what is drawn follows the cockpit's one currency reading (GoldenPathEnvelope): current in the
signal, withheld dim as the last known good route, the rest nothing. Hovering an item names it
through the engine's canvas mouse path. Goldens for the currency words in both skins plus the
hover; the shipped dock document and the view-topology table follow."
- 2026-09-25T18:32:09Z @neo-fable-clio referenced in commit `f83eacf` - "feat(cockpit): the Route graph pane draws the Golden Path on the canvas worker (#213)

A south-strip pane binds the cockpit provider's goldenPathEnvelope leaf and hands the landed
envelope to a canvas-worker renderer: the computed route as a ranked spine, its citations as
satellites, a citation shared by several items placed once with an edge to each, the currency
line above. The layout is a pure module unit-tested without a canvas; the renderer only draws;
what is drawn follows the cockpit's one currency reading (GoldenPathEnvelope): current in the
signal, withheld dim as the last known good route, the rest nothing. Hovering an item names it
through the engine's canvas mouse path. Goldens for the currency words in both skins plus the
hover; the shipped dock document and the view-topology table follow."
- 2026-09-25T18:57:46Z @tobiu referenced in commit `4bffce2` - "Merge pull request #216 from neomjs/feat/213-golden-path-graph

feat(cockpit): the Golden Path graph pane draws the route on the canvas worker (#213)"
- 2026-09-25T18:57:46Z @tobiu closed this issue

