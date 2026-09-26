---
id: 230
title: 'Observatory pane: the Golden Path route as a 3D WebGL2 scene'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-26T07:28:21Z'
updatedAt: '2026-09-26T07:28:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/230'
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
---
# Observatory pane: the Golden Path route as a 3D WebGL2 scene

## Context

The operator's 2026-09-26 goal: the Golden Path textual view AND the 3D graph inside the Fleet Manager, with follow-ups expected (gravity wells, a roster column that filters and highlights one peer's items, highlighting the golden path, a node selection model with details on zoom). D#19151 (neomjs/neo#19151) graduated H3 — the graph observatory — for its bounded first slice (Grace's `[GRADUATION_APPROVED]` DC_kwDODSospM4BG8bU, 2026-09-25 10:19Z): the Golden-Path-led focus lens (H3-d) first, H3-e's scene model second, the team lens (H3-g) adopted, the engine-native renderer (H3-a: WebGL2 in the canvas worker, foveated LOD) as the renderer shape — measured on 100k nodes / 1M edges at far 118 · mid 120 · near 116 fps in the canvas worker. The graduation shape's Institution sub is "the focus lens, then multiscale, with the team lens" under neomjs/neo#10034 promoted to an epic; the Brain feed is neomjs/neo-agent-brain#533, the team-lens projection neomjs/neo-agent-brain#534.

## The Problem

The cockpit has two Golden Path readings on `dev` — the text pane (#210 → #215) and the 2D route graph (#213 → #216), both bound to the cockpit's `goldenPathEnvelope` leaf. Neither is the observatory: a navigable 3D scene the operator orbits and zooms, in which the route is the golden line through the graph the institution steers by. The bounded scene feed (neomjs/neo-agent-brain#533) does not exist yet; the pane must not wait for it, and it must not invent data while it waits.

## The Architectural Reality

- `apps/agentos/canvas/GoldenPathGraph.mjs` (2D, the default `'2d'` context, one frame per change, no loop), `apps/agentos/view/fleet/goldenpath/GraphCanvas.mjs` (the `Neo.app.SharedCanvas` host: the envelope forwarded as one plain worker message, the local `mousemove`, the offset-pair pointer fix) and `GraphContainer.mjs` (currency head + canvas; the canvas only where `Neo.config.useCanvasWorker && !unitTestMode`) — the pane shape this ticket lifts.
- `apps/agentos/util/GoldenPathGraphLayout.mjs` (the pure 2D layout) and `apps/agentos/util/GoldenPathEnvelope.mjs` (the one derivation: `currency()`, `routeOf()`, `citationLabel()`).
- Engine seams on `dev` for a WebGL renderer in the canvas worker: `Neo.canvas.Base` `contextType` / `contextAttributes` (neomjs/neo#19169); the host forwards wheel, drag and modifiers to `onWheel` / `onMouseDown` / `onMouseUp` and `mouse.dx/dy` (neomjs/neo#19173); `updateSize` with the host's pixel ratio.
- The D#19151 canvas-worker witness (archived outside the repo; its numbers are in the Discussion body): a `Neo.canvas.Base` subclass with `contextType: 'webgl2'`, one program (points and lines, `gl_PointSize` by depth, additive blend), one VAO per layer (nodes, edges, bundles, centroids, route), a yaw / pitch / distance camera, orbit from `mouse.dx/dy` while button 1 is held, wheel → distance. That renderer's shape is what this pane ports; its synthetic scene generator is not.
- The workspace build keeps only `src/canvas/…` of the engine inside the package (neomjs/neo#19165): the renderer lives in `apps/agentos/canvas/` for this slice; its extraction into a `src/canvas` primitive is the epic's engine leaf, after this slice proves the shape.
- Cockpit panes are declared in `apps/agentos/view/fleet/cockpit/Container.mjs` `panes` (:141; "Golden Path" :204, "Route graph" :211); the design system (#13) governs the pane chrome, the visual baseline harness (#11) its goldens.

## The Fix

1. `apps/agentos/canvas/Observatory.mjs` — `Neo.canvas.Base`, `contextType: 'webgl2'`, `contextAttributes: {antialias: false, powerPreference: 'high-performance'}`; remote: `setScene`, `setTheme`, `updateMouseState`, `updateSize`, `getStats` (+ the base lifecycle set). Frames only while the pointer moves the camera or a scene lands — idle draws nothing. Layers: edges (lines, dim), nodes (round points, sized by score / degree), the route (a golden line strip + points). Currency: `current` = the route in the signal gold, `withheld` = dim ink, anything else = a cleared surface.
2. `apps/agentos/util/ObservatorySceneLayout.mjs` — pure, unit-tested: turns a scene envelope into typed arrays (positions, colours, sizes, edge indices, route indices). The scene envelope shape is declared here (`{nodes: [{id, kind, label, score?, degree?, peer?}], edges: [[a, b]], route: [id…], counts, budget, completeness}`) and `fromGoldenPath(envelope)` derives this slice's only scene from the Golden Path envelope: route items on a helix by rank, each item's citations on a ring around it — so neomjs/neo-agent-brain#533 replaces the derivation, never the renderer.
3. `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs` + `ObservatoryCanvas.mjs` — the pane (currency head, the hovered node's label in the head, the canvas), registered beside "Route graph" as "Observatory"; the host lifts `GraphCanvas`'s pointer handling and adds wheel + drag forwarding (neomjs/neo#19173's shape) so the worker orbits and zooms.
4. Specs: the layout's unit spec (route order, citation rings, currency); a Neural Link e2e arm that lands a fixture envelope through `writeGoldenPath` and reads the renderer's `getStats()` (frames > 0, the camera moved after a forwarded drag and wheel); goldens for `current` and `withheld` in both themes through #11 (the surface is deterministic per fixture and camera).
5. Every file under the 1k line bar; the shaders are string constants inside the renderer.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.canvas.Observatory` remote `setScene({scene})` | this ticket (the scene shape in `ObservatorySceneLayout.mjs`) | draws the scene's nodes, edges and route | `null` clears the surface | JSDoc | AC-1, AC-4 |
| `ObservatorySceneLayout.fromGoldenPath(envelope)` | `GoldenPathEnvelope` (#215) | route items + citations → a scene | an empty scene for any currency but `current` / `withheld` | JSDoc | AC-1 |
| pane "Observatory" in `cockpit/Container.mjs` `panes` | #133's declared panes | a south-strip tab beside "Route graph" | — | #13 | AC-3 |

## Decision Record impact

aligned-with ADR 0029 (the pane is a declared pane and tears out like any other). Decision Record (D#19151): `Optional` — H3 takes an ADR only if Brain-computed coordinates become a served projection (OQ5's re-entry), not in this slice.

## Acceptance Criteria

- [ ] AC-1 With a `current` Golden Path fixture the pane draws the route as a golden line strip through its items and every citation as a node on its item's ring; with `withheld` the same in dim ink; with any other currency an empty surface — the layout's unit spec per state, the renderer's `getStats()` read in the NL arm.
- [ ] AC-2 Drag orbits and wheel zooms the scene in the browser and the packaged shell (NL arm: the camera state changes after a forwarded drag and wheel); idle draws no frames (`getStats().frames` stable across one idle second).
- [ ] AC-3 "Observatory" is a declared pane beside "Route graph", tears out with the cockpit's mechanics, and wears the FM tokens (#13); goldens for `current` and `withheld` in both themes pass the #11 harness.
- [ ] AC-4 The scene shape and `setScene` are the one entry a Brain scene feed lands in — reviewed against the diff; the feed itself is neomjs/neo-agent-brain#533.
- [ ] AC-5 (post-merge) The operator orbits the real route in the team shell (plane-attach, after #228).

## Out of Scope

The bounded scene feed (neomjs/neo-agent-brain#533); the multiscale LOD over that feed (H3-e — the next Institution leaf; the witness's far / mid / near lap is its receipt); the team lens (H3-g: the roster column, colour by peer, "working on now" avatars — the operator's "filter or highlight all items of a peer"; needs neomjs/neo-agent-brain#534); the node selection model with details on zoom; gravity wells; the engine `src/canvas` primitive + `CANVAS_EXPECTATIONS` (the epic's engine leaf). Each is its own leaf once this slice lands.

## Avoided Traps

- A library renderer on the main thread (H3-b, rejected in D#19151: the operator's "our own, on OffscreenCanvas + Neo"; the main thread stays free beside the App Worker).
- Synthetic placeholder data while the feed is missing — the pane synthesizes nothing (#210 AC-4's law); the Golden Path envelope is real data and the only scene of this slice.
- Labels on the GL surface — text in WebGL needs a glyph atlas; the hovered label renders in the pane's DOM head (the `Neo.app.header.Canvas` precedent: DOM stays real, the canvas flows around it).
- A render loop that never sleeps — the 2D graph's one-frame-per-change law holds; a frame is owed only to a camera move or a scene change.

## Related

neomjs/neo#10034 (the epic), neomjs/neo#19151 (H3-a/d/e/g, OQ5 deferred), #213 / #216 (the 2D graph), #210 / #215 (the envelope), #228 (the pin that makes the live envelope reachable), neomjs/neo#19169, neomjs/neo#19173 (the engine seams), neomjs/neo-agent-brain#533 (the feed), neomjs/neo-agent-brain#534 (the lens' node facts), #8 (the COP the observatory serves), #13, #11.

Live latest-open sweep: the latest 20 open issues of this repository (07:18:32Z), of neomjs/neo and neomjs/neo-agent-brain (07:23:53Z) — no equivalent. A2A in-flight sweep 07:18–07:24Z: no claim on the observatory (Grace = the tear-out lane, Ada = #227, Mnemosyne = neo #19232). Epic-layer sweep: every open epic's `Terminal predicate:` line across the three repos — none names the observatory (neomjs/neo#15239 is docking, neomjs/neo-agent-brain#122 the route producer, #8 the COP it serves). Memory Core sweep (`query_raw_memories`): Emmy's D#19151 cycle (2026-09-24) — the Discussion, no ticket. Own-assignment sweep: none of mine covers it. Structure map: N/A (the `apps/agentos` siblings name the placement: `canvas/`, `util/`, `view/fleet/goldenpath/`).

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("observatory pane WebGL2 canvas worker Golden Path 3D scene orbit")`

## Timeline

- 2026-09-26T07:28:21Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-26T07:28:23Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T07:28:23Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T07:28:23Z @neo-fable-clio added the `ai` label
- 2026-09-26T07:28:23Z @neo-fable-clio added the `design` label
- 2026-09-26T07:28:54Z @neo-fable-clio added parent issue #10
- 2026-09-26T07:30:21Z @neo-fable-clio cross-referenced by #10034

