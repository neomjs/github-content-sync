---
id: 288
title: 'Observatory draws the whole graph in clusters, the route as an overlay'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T12:26:29Z'
updatedAt: '2026-09-27T15:52:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/288'
author: neo-opus-grace
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
closedAt: '2026-09-27T15:52:32Z'
---
# Observatory draws the whole graph in clusters, the route as an overlay

## Context
The Observatory row of D#19151's recovery map (neomjs/neo#19151, comment 18624317; this row proposed in 18624407 and accepted into the lead's fold). Tobi's ruling: the Observatory in FM is the demonstrated 100k-node experience, meaning three LOD levels, wheel zoom and drag rotation, with the Golden Path as an optional overlay. A capped neighbourhood around the route does not fulfil it.

## The Problem
The FM observatory draws only the route's neighbourhood, and it draws that without the renderer's level of detail:
- `Observatory#ink` hands the engine `{colors, edges, positions, sizes}` and never passes `clusters`. `Neo.canvas.GraphScene` draws its far/mid/near tiers only for a scene that carries them, so `getLodLevel()` stays `'full'`.
- `ObservatorySceneLayout` puts the route's seeds on a helix and rings every other node around its nearest seed. The route *is* the scene, so it can't be switched off. A node the route doesn't reach has no meaningful place.
- The landing seam exists but has only carried 7 nodes. `ReadingSurfacesController#writeGraphScene` is the live read's own WRITE, and `FleetObservatoryNL`'s `land` uses it for a 7-node fixture. Nothing has pushed whole-graph scale through it.
- Three steps don't scale:
  - the pane deep-clones the provider proxy with `JSON.parse(JSON.stringify())`;
  - the layout keys every edge by `JSON.stringify`;
  - the node list is a `Neo.list.Base` with one DOM row per node.

Wheel zoom and drag rotation already work: `ObservatoryCanvas` forwards moves, buttons and a non-passive wheel to `GraphScene`'s orbit camera.

## The Architectural Reality
- `apps/agentos/util/ObservatorySceneLayout.mjs` `fromGraphScene`: the envelope → `{nodes, edges, seeds, index, …}` with helix positions.
- `apps/agentos/canvas/Observatory.mjs` `ink`, `pick`, `locate`, `setScene`: the canvas-worker renderer, extending `Neo.canvas.GraphScene`.
- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs` `afterSetEnvelope`: the pane. It binds the provider's `graphSceneEnvelope` leaf and feeds the canvas and the two lists.
- Engine (pin 942b43c): `GraphScene` takes `clusters` (a `Uint32Array`, one id per node) and `paths` (`Uint32Array[]`), and has `lod`, `lodLevel` and `getLodLevel()`. Its scale harness `examples/component/graphScene/demoScene.mjs` `createClusteredScene` places 64 clusters on a golden-angle sphere.
- `apps/agentos/view/fleet/cockpit/ReadingSurfacesController.mjs:269` `writeGraphScene`: `GraphSceneEnvelope.fromWire` → the provider's `graphSceneEnvelope` leaf. The live `loadGraphScene` read ends in it, and `FleetObservatoryNL`'s `land` calls it.

## The Fix
1. **One layout for every read.** Communities come from the read's own edges: label propagation with id-ordered tie-breaks, so it's deterministic. Communities sit on a golden-angle sphere, and each node takes a hashed offset inside its community's shell. Positions depend only on ids, edges and communities. The helix retires.
2. **The route becomes an overlay.** `ink` passes `clusters`, colours nodes by community, and, only while the overlay is on, draws the route as a `GraphScene` path with its seeds highlighted. The toggle re-inks. Positions, `index` and the selection don't move.
3. **Scale.** The canvas gets typed arrays plus the ids `pick` needs, not node objects. The pane reads plain data without a JSON round trip. The lists show the route and the selection's neighbourhood, not one row per node.
4. **The scale witness reuses `writeGraphScene`.** A generated fixture has 100k nodes, 64 communities, about 300k edges (mostly intra-community) and a 10-seed route. It lands through the live read's WRITE and crosses envelope admission, the provider leaf, the pane, the layout and the canvas worker. It does not cross the Fleet bridge read, which is the live-input row.

## Contract Ledger Matrix
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The observatory scene (pane → canvas worker) | `ObservatorySceneLayout.fromGraphScene` | Clustered positions, `clusters`, the route as a path while the overlay is on. Positions stable across the toggle and across row order | An unavailable or empty read draws nothing (unchanged) | The layout's and renderer's JSDoc | AC-1–AC-3 |
| The pane's node and relation lists | `ObservatoryContainer` | The route and the selection's neighbourhood at any scale | An empty read lists nothing (unchanged) | The pane's JSDoc | AC-4 |

## Acceptance Criteria
- [x] AC-1: Every read lays out in communities derived from its edges. The same read in any row order lays out identically, and a two-community read separates its communities (unit).
- [x] AC-2: `ink` passes `clusters`. On a clustered scene, `getLodLevel()` moves from far through mid to near as the camera closes in (unit on the renderer; NL through the FM window with wheel events).
- [x] AC-3: The overlay toggle draws the route as a path and highlights its seeds only while on. `locate(id)`, the scene's `snapshotId` and the selection are unchanged across toggles (unit + NL).
- [x] AC-4: At 100k nodes the node and relation lists hold the route and the selection's neighbourhood, never one row per node (unit).
- [x] AC-5: The FM entry carries the fixture. The 100k envelope lands through `writeGraphScene`, and the NL spec proves AC-2 and AC-3 in the FM window, recording the landing-to-first-frame time. This is a fixture receipt only. D#19151's live whole-graph input row, Ada's, stays open.
- [x] AC-6: The existing observatory arms pass: selection, `pick`, `locate`, list parity and `FleetObservatoryNL`. Assertions on the helix's shape are re-expressed, and goldens are re-captured over the new layout.

## Out of Scope
- The live whole-graph input: the producer, budgets, transfer and layout ownership (D#19151's reopened row, Ada's). The fixture proves the FM side only.
- The Golden Path content reader (the lead's lane).
- A wire field for communities. Whoever owns the producer decides whether one ships.

## Avoided Traps
- **Two layouts, helix for small reads and clusters for large ones.** That makes two observatories behind one pane. The ruling is one graph with an optional route.
- **Node objects across the worker boundary at 100k.** Structured-cloning 100k objects per scene is the cost the engine's typed-array scene exists to avoid.
- **Closing the live row with the fixture.** A generated graph certifies the renderer path, not the Brain's data.

## Related
#10, neomjs/neo#19151, neomjs/neo#19302, neomjs/neo#19295, #272, #273, #266

Decision Record impact: none.

Live latest-open sweep: latest 20 open issues read at 2026-09-27T12:25:14Z and again before filing; no equivalent. The nearest are #244 (a Home canvas) and #287 (golden re-captures). Lexical "observatory": #247, #244, #10, #8 open, and #258 closed (the bounded slice this supersedes). "whole graph": none. A2A claim sweep (30 newest, all read states): Ada yielded this integration row and self-selected the live producer/layout row; no other claim. MC sweep ("observatory capped neighbourhood helix route graph timeline rejected whole graph clusters LOD FM pane"), 6 results: D#19151's history and the lead's recovery record accepting this row. Own-assignment sweep: #285, #278, #11, none overlapping.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6
Retrieval Hint: "FM observatory whole graph clustered layout LOD route overlay 100k fixture"

Authored by Grace (Claude Opus 5.5, Claude Code).



## Timeline

- 2026-09-27T12:26:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T12:26:31Z @neo-opus-grace added the `enhancement` label
- 2026-09-27T12:26:31Z @neo-opus-grace added the `agent-os` label
- 2026-09-27T12:26:31Z @neo-opus-grace added the `ai` label
- 2026-09-27T13:10:51Z @neo-opus-grace cross-referenced by PR #291
- 2026-09-27T13:15:46Z @neo-opus-grace cross-referenced by #19311
- 2026-09-27T13:19:22Z @neo-opus-grace cross-referenced by PR #19312
- 2026-09-27T14:18:44Z @tobiu referenced in commit `a9f0ee4` - "feat(agentos): the observatory glows — dust-sized nodes that add up on the dark skin, the route in its own gold, the fitted camera on every node (#288)"
- 2026-09-27T14:28:36Z @neo-opus-ada cross-referenced by #583
- 2026-09-27T14:42:00Z @tobiu referenced in commit `d426c65` - "feat(agentos): the observatory lays out the whole graph in communities and draws the route as an overlay (#288)

Every graph read now lays out by modularity communities (Louvain over the
read's own edges, deterministic in id order) on a golden-angle sphere, each
node at a point its id draws inside its community's ball, so positions
depend only on ids and edges and the route never moves a node. The renderer
inks by community, hands a large scene's communities to GraphScene as
clusters so its far/mid/near level of detail engages, and draws the route as
a toggleable path overlay; a selection or overlay change re-inks in the
canvas worker without the scene crossing again. The pane reads the provider
leaf through a closed-shape projection instead of a JSON round trip, and its
lists keep to a budget with titles that say how many of how many."
- 2026-09-27T14:42:00Z @tobiu referenced in commit `9284d06` - "test(agentos): the observatory's NL arms read the community and path counts, and a 100k whole graph lands through the live read's write (#288)"
- 2026-09-27T14:42:00Z @tobiu referenced in commit `95d1d9a` - "feat(agentos): the observatory reads at whole-graph scale — muted community hues, density-scaled nodes, a finer route ribbon, a quiet Route toggle (#288)

Every community takes a muted golden-angle hue from the signal's, so
communities read apart while the route stays the brightest ink; nodes shrink
with the read's density so 100k nodes read as a cloud; the route draws as a
finer ribbon and keeps its colour under a selection; the Route toggle is the
roster's quiet ghost verb. The visual arm captures the route-off state and
parks the pointer so no tooltip lands in a shot, and the scale arm attaches
far/near/route-off pictures of the 100k graph for review."
- 2026-09-27T14:42:00Z @tobiu referenced in commit `c386c97` - "feat(agentos): the observatory glows — dust-sized nodes that add up on the dark skin, the route in its own gold, the fitted camera on every node (#288)"
- 2026-09-27T15:07:39Z @neo-opus-ada cross-referenced by PR #587
- 2026-09-27T15:20:05Z @tobiu referenced in commit `0f3a5bb` - "feat(agentos): the observatory lays out the whole graph in communities and draws the route as an overlay (#288)

Every graph read now lays out by modularity communities (Louvain over the
read's own edges, deterministic in id order) on a golden-angle sphere, each
node at a point its id draws inside its community's ball, so positions
depend only on ids and edges and the route never moves a node. The renderer
inks by community, hands a large scene's communities to GraphScene as
clusters so its far/mid/near level of detail engages, and draws the route as
a toggleable path overlay; a selection or overlay change re-inks in the
canvas worker without the scene crossing again. The pane reads the provider
leaf through a closed-shape projection instead of a JSON round trip, and its
lists keep to a budget with titles that say how many of how many."
- 2026-09-27T15:20:05Z @tobiu referenced in commit `6e81927` - "test(agentos): the observatory's NL arms read the community and path counts, and a 100k whole graph lands through the live read's write (#288)"
- 2026-09-27T15:20:05Z @tobiu referenced in commit `80e3e2e` - "feat(agentos): the observatory reads at whole-graph scale — muted community hues, density-scaled nodes, a finer route ribbon, a quiet Route toggle (#288)

Every community takes a muted golden-angle hue from the signal's, so
communities read apart while the route stays the brightest ink; nodes shrink
with the read's density so 100k nodes read as a cloud; the route draws as a
finer ribbon and keeps its colour under a selection; the Route toggle is the
roster's quiet ghost verb. The visual arm captures the route-off state and
parks the pointer so no tooltip lands in a shot, and the scale arm attaches
far/near/route-off pictures of the 100k graph for review."
- 2026-09-27T15:20:05Z @tobiu referenced in commit `734a01e` - "feat(agentos): the observatory glows — dust-sized nodes that add up on the dark skin, the route in its own gold, the fitted camera on every node (#288)"
- 2026-09-27T15:39:54Z @tobiu referenced in commit `0ad7334` - "fix(agentos): the observatory scene crosses to the canvas worker as typed arrays by index, the node list keeps one extra row, the 100k receipt waits for its scene's frame (#288)"
- 2026-09-27T15:45:34Z @tobiu referenced in commit `e4974bd` - "feat(agentos): the observatory lays out the whole graph in communities and draws the route as an overlay (#288)

Every graph read now lays out by modularity communities (Louvain over the
read's own edges, deterministic in id order) on a golden-angle sphere, each
node at a point its id draws inside its community's ball, so positions
depend only on ids and edges and the route never moves a node. The renderer
inks by community, hands a large scene's communities to GraphScene as
clusters so its far/mid/near level of detail engages, and draws the route as
a toggleable path overlay; a selection or overlay change re-inks in the
canvas worker without the scene crossing again. The pane reads the provider
leaf through a closed-shape projection instead of a JSON round trip, and its
lists keep to a budget with titles that say how many of how many."
- 2026-09-27T15:45:34Z @tobiu referenced in commit `41b4dce` - "test(agentos): the observatory's NL arms read the community and path counts, and a 100k whole graph lands through the live read's write (#288)"
- 2026-09-27T15:45:35Z @tobiu referenced in commit `7a56a84` - "feat(agentos): the observatory reads at whole-graph scale — muted community hues, density-scaled nodes, a finer route ribbon, a quiet Route toggle (#288)

Every community takes a muted golden-angle hue from the signal's, so
communities read apart while the route stays the brightest ink; nodes shrink
with the read's density so 100k nodes read as a cloud; the route draws as a
finer ribbon and keeps its colour under a selection; the Route toggle is the
roster's quiet ghost verb. The visual arm captures the route-off state and
parks the pointer so no tooltip lands in a shot, and the scale arm attaches
far/near/route-off pictures of the 100k graph for review."
- 2026-09-27T15:45:35Z @tobiu referenced in commit `7edf6f6` - "feat(agentos): the observatory glows — dust-sized nodes that add up on the dark skin, the route in its own gold, the fitted camera on every node (#288)"
- 2026-09-27T15:45:35Z @tobiu referenced in commit `a48c828` - "fix(agentos): the observatory scene crosses to the canvas worker as typed arrays by index, the node list keeps one extra row, the 100k receipt waits for its scene's frame (#288)"
- 2026-09-27T15:52:32Z @tobiu referenced in commit `f2cb97f` - "Merge pull request #291 from neomjs/grace/288-observatory-whole-graph

feat(agentos): the observatory draws the whole graph in clusters with the route as an overlay (#288)"
- 2026-09-27T15:52:32Z @tobiu closed this issue
- 2026-09-27T16:20:02Z @neo-opus-grace cross-referenced by #305
- 2026-09-27T16:23:09Z @neo-opus-grace cross-referenced by PR #306

