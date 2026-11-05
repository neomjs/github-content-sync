---
id: 839
title: Observatory wells read as unlabeled dust on the installed build
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:07:20Z'
updatedAt: '2026-11-05T12:54:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/839'
author: neo-opus-vega
commentsCount: 4
parentIssue: 312
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-05T12:54:49Z'
---
# Observatory wells read as unlabeled dust on the installed build

## Context

Row 3's installed sitting (#485, 2026-10-30 19:4x–21:0xZ; installed Neo Harness: Institution `20d89be` / Brain `1bf8bd91` / engine `477b65bd9e`; the operator's display at 1992×1290) ran check 3 of the packet (#312, comment 5966703991): *at the operator's display and skin, no zoom, name three wells from their labels and tell the halo from the wells.* **The operator could not.** With the route off his frame showed dust only: no labels, no edges, no visible wells. With the route on, the gold Golden Path polyline is the only legible structure; the wells are the same dust behind it. Halo nodes are a few pixels at every available zoom and cannot be clicked by hand (known to the operator; no ticket found by title or body search, so it is recorded here, not re-filed).

The scene carried everything a legible drawing needs, read over the Neural Link during the sitting: 48 wells with labels (wells[0] `MailboxService`, 122 nodes at the cap; wells[1] `A2A Handoff Protocol`; wells[2] `#13624`), `haloFrom` 48, 132,682 nodes in the halo, completeness `truncated`, out of a current read of 134,050 nodes and 21,139 edges (captured 19:43:52Z, 30 s after the window).

## The Problem

The geography says nothing on its own. A well is a density of dust with no name, no boundary and no edges; the halo is the same dust further out. The one thing that reads is the route overlay, which is not the geography. #310 shipped "readable wells" at L2 (fixtures, unit and NL arms) and deferred readability at the real display to this installed check, which now fails.

## The Architectural Reality

- `apps/agentos/util/ObservatorySceneLayout.mjs` (`fromGraphScene`) computes the wells (label, size, cap) and the halo shell (`haloGeometry`); the canvas worker (`AgentOS.canvas.Observatory`, `apps/agentos/canvas/Observatory.mjs`) draws from that scene.
- The head (`ObservatoryHeadComponent`) carries counts, never well names; the side panel (`ObservatoryViewContainer`, `ObservatoryTeamContainer`, `ObservatoryNodesHeadContainer`, `ObservatorySelectionContainer`) has no well list, so a well's name exists only in the scene data.
- Design authority: #310 ("The Observatory draws readable wells", AC-6 "label and halo readability") makes readability the intended behavior; no record makes label-less wells intended (searched: #310, #320, #333 bodies, `Viewport.mjs` and the goldenpath JSDoc).

## The Fix

A design read first (the designed-surface gate: #505 is the views umbrella), then one PR:

1. Draw each well's label at its centroid, sized by the well's node count, with the halo drawn one step dimmer than the wells so the two read as different things.
2. A well list in the side panel (name · size · over-cap mark), selecting a well focusing the camera on it, so a name is also reachable without reading the canvas.
3. The PR body carries a BEFORE/AFTER screenshot pair at 1992×1290, route on and route off.

## Acceptance Criteria

- [ ] On the installed build at the operator's display, route on and route off, the operator names three wells from drawn labels; the receipt is his frame on #485.
- [ ] Wells and halo are two visibly different treatments in the same frame.
- [ ] An NL arm reads the drawn labels of the top-N wells from the canvas (`readStats()` or an equivalent read) and goes red on `dev` first.
- [ ] A well list exists in the side panel and selecting a row focuses that well without moving any node.
- [ ] BEFORE/AFTER screenshot pair in the PR body.

## Out of Scope

The zoom that cannot enlarge a region (the few-pixel halo nodes stay unclickable): a sighting on #505, not this leaf. Node hit targets. The route overlay's own drawing.

## Related

Parent: #312. Receipt: #485 (the 2026-10-30 sitting). #310 AC-6 (the deferred check). #505 (design umbrella; the sighting that actions and labels share one ghost style, and the peer palette repeating after five peers, are recorded there). #320 AC-6 (the halo split).

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-30T21:06:46Z; no equivalent (#505/#507 are design umbrellas, not this failure). A2A sweep: last 30 `lane-claim` messages, none on the Observatory. MC sweep: "Observatory wells unreadable no labels dust halo nodes tiny zoom readability defect", 5 results, no prior decision. Own-assignment sweep: 4 open (#732, #651, #649, #485), none overlapping.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "row 3 sitting check 3 wells unreadable dust no labels installed Observatory"

## Timeline

- 2026-10-30T21:07:20Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T21:07:22Z @neo-opus-vega added the `bug` label
- 2026-10-30T21:07:22Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T21:07:22Z @neo-opus-vega added the `ai` label
- 2026-10-30T21:07:22Z @neo-opus-vega added the `design` label
- 2026-10-30T21:09:10Z @neo-opus-vega added parent issue #312
- 2026-10-30T21:10:05Z @neo-opus-vega cross-referenced by #485
- 2026-10-30T21:10:07Z @neo-opus-vega cross-referenced by #505
- 2026-10-30T21:10:45Z @neo-opus-vega cross-referenced by #312
### @neo-opus-vega - 2026-11-04T19:38:01Z

### Journey delta, for the design read this ticket names before its PR opens

**Today**, at the sitting's 134,050 nodes, every well node draws at the renderer's smallest size (`nodeMin`) and every halo node at 0.6 of it. The 48 wells hold 1,368 nodes; the halo shell around them holds 132,682. On the dark skin points add their light, so the shell glows in front of the wells. Nothing on the canvas names a well: the names exist only in the scene data.

**After the PR:**
1. **Each well is named on the canvas.** The label sits at the well's centre (its anchor node), sized by the well's node count, in the skin's ink with a ground-coloured outline so it reads over dust. A label that would overlap a larger well's label is left out of that frame, so the largest wells always read and smaller ones appear as the camera nears them. The canvas worker draws the labels in the same frame as the points, so they follow an orbit without lag.
2. **Wells and halo get two treatments.** Well nodes are sized from the wells' own count, not the whole read's, so a well reads as a body of points. The halo stays dust, inked one tier dimmer (`lineSoft` where it takes `line` today), so it recedes on both skins.
3. **A Wells section in the side panel**, between Team and Nodes, collapsing like them. Each row shows the name, the node count, and "full" when the cap stopped the well. The largest well comes first. Selecting a row turns the camera to that well: target and distance only. No node moves and the graph selection stays.

Unchanged: the route overlay, the lens, the heat and picking. Out of scope stays as filed.

**Already read:** the label layer and fly-to follow the design seat's zoom-ladder read on #312 (6222093228). That read names the wells at the sky level, caps labels at 30 on screen, and sets 11 px as the smallest. The engine half is neomjs/neo#20286. What this delta still needs read is the halo and well treatment (2) and the Wells section (3).

**Two calls I'm making unless the read says otherwise:**
- Every well is a label candidate, largest first, decluttered per frame, with at most 30 on screen.
- Team stays the default open section; Wells opens only when chosen.

Either the design seat or the operator can do the read. I'll add a prototype frame at 1992×1290 here, route on and route off, once it draws.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-11-04T19:41:58Z @neo-opus-vega cross-referenced by #20286
- 2026-11-04T19:54:48Z @neo-opus-vega cross-referenced by PR #20288
### @neo-opus-vega - 2026-11-04T19:59:17Z

### Handover: state on 2026-11-04 at 20:0xZ (Vega, solo-refresh sunset at the context gate)

**State**
- **Design.** The journey delta is 6640247392 above. The design seat already read the label layer and fly-to on #312 (6222093228). The halo and well treatment and the Wells section still wait for a read. I asked Clio (A2A 973f8dc0); she's away today, and the operator's read would clear the gate too. This ticket's PR does not open before that read.
- **Engine half.** neomjs/neo#20286 became PR neomjs/neo#20288. It adds the `GraphScene` label atlas, the static `placeLabels`, `flyTo`, and `getStats().labels.drawn` plus `flying`. Its new arms went red on `dev` first (6 failed, 21 passed); on the branch 46 unit and 3 e2e pass, the e2e covering the fly-to and a context restore. Euclid is the primary reviewer. The Institution tracks `neo.mjs#dev`, so this ticket's PR goes green only after #20288 merges and the dependency moves.

**Plan for this ticket's one PR**
1. `ObservatorySceneLayout.wire()` carries the wells as `wells: {nodes, sizes, texts}`. `nodes` holds the hub indices through `scene.index`, and `texts` takes the label, or the id where there is none.
2. `AgentOS.canvas.Observatory.ink()`:
   - Labels come from the wells, largest first, at 11–15 px by node count, in `palette.ink` over a `palette.panel` outline.
   - Halo nodes take `palette.lineSoft`.
   - Well nodes are sized from the wells' own count (`count` minus the halo); the halo keeps the whole read's size.
3. `ObservatoryCanvas` gets `flyTo(id)`, which calls `renderer.flyTo({index})`. `readStats()` maps `labels.drawn[].index` to ids.
4. The Wells section sits between Team and Nodes:
   - `AgentOS.model.ObservatoryWell` and `AgentOS.store.ObservatoryWells`, pane-owned like `GraphSceneNodes`.
   - `ObservatoryWellList extends Neo.list.Base`: each row shows name, size, and "full" at `wellCap`.
   - A section head, and `openSection` gains `wells`.
   - Selecting a row calls `flyTo`. Only the camera moves; the graph selection stays.
5. An NL arm in `FleetObservatoryNL.spec.mjs` on a `wholeGraphEnvelope` fixture, red on `dev` first:
   - The largest well's name is the first label drawn, and no more than 30 labels draw.
   - The Wells list leads with the largest well.
   - A row flies its hub to the canvas centre, with `snapshotId` and `selectedId` unchanged.
   The local `FleetCockpitVisual` goldens will move with the halo ink and the labels.
6. A BEFORE/AFTER pair at 1992×1290, with the route on and off.

**Before writing app code**, load `src/Neo.mjs`, `core/Base`, `state/Provider`, `data/Model` and `data/Store`, plus the siblings `GraphSceneNodes`, `GraphSceneNode` and `ObservatoryNodeList`. Until #20288 lands, trial it locally by copying that branch's `src/canvas/GraphScene.mjs` into `node_modules/neo.mjs` (untracked); `npm ci` restores the original.

The PR is due Fri Nov 6 (my promise to Ada) so that row 3's checks 3 and 7 can re-run before the Nov 13 gate.

— Vega (Opus 5.5, Claude Code) 🌿


### @neo-opus-vega - 2026-11-04T23:37:01Z

### Prototype frames for the design read (delta 6640247392)

1992×1290, dark skin. The scene is a synthetic fixture with 48 density wells and a 30,000-node halo; provenance is in the [media README](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/README.md). It is not the plane's data. The operator's frame on the installed build stays AC-1's receipt.

| BEFORE · route on | AFTER · route on |
|---|---|
| ![before, route on](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/before-route-on.png?raw=true) | ![after, route on](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/after-route-on.png?raw=true) |
| **BEFORE · route off** | **AFTER · route off** |
| ![before, route off](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/before-route-off.png?raw=true) | ![after, route off](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/after-route-off.png?raw=true) |
| **AFTER · Wells open** | **AFTER · a row flown to** (the third well's hub at the centre; the selection stays) |
| ![after, wells open](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/after-wells-open.png?raw=true) | ![after, flown to the third well](https://github.com/neomjs/neo-agent-institution/blob/496425b136ef9b97d3edb98cc9d72d1e928706e2/839-observatory-well-labels/after-flown-to-third-well.png?raw=true) |

**Changed since the delta**
- A label now sits 6 px above its hub instead of centred on it. The first frame showed each name covering the node it names, and the gold route beacons with it. The fix is in the engine half, neomjs/neo#20288 at `ecb09dda10`, as `labelStyle.lift`.
- Wells is its own section container (`ObservatoryWellsContainer`, the way Team is built). Inlining it pushed the pane past the app file-size bar.

**Two more calls I'm making unless the read says otherwise**
- Labels keep their ink under a selection, the lens or the heat. They name the geography, not the focus.
- A well's row stays marked after its flight, so the list shows where the camera went.

**State.** Branch `vega/839-observatory-well-labels` at `0b6cdc6` is pushed, with no PR yet.
- Green locally against engine dev with the #20288 GraphScene: unit 2107 passed (0 failed), Observatory NL 8/8, the 14 Observatory goldens re-captured and inspected, the stamp re-written.
- The new arms went red on dev first.
- The PR opens after this read. Its lock moves to the engine revision #20288 merges as, because CI's locked-dependency unit run needs the label layer.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-11-05T11:28:44Z @neo-opus-vega referenced in commit `171cdda` - "feat(agentos): the Observatory names its wells on the canvas and in a Wells section (#839)

The wire scene carries the wells (hub index, size, name). The renderer labels
them largest first, 11-15 px by size, in the skin's ink over its ground. Halo
dust takes lineSoft, and well nodes are sized from the wells' own count, so a
well reads as a body of points. A Wells section between Team and Nodes lists
them, "full" at the strategic cap. A row turns the camera to its hub through
ObservatoryCanvas.flyTo; no node moves and the selection stays. readStats
names each drawn label's node by id.

Needs the GraphScene label layer and flyTo of neomjs/neo#20288."
- 2026-11-05T11:28:44Z @neo-opus-vega referenced in commit `1d694a0` - "build(agentos): the lock resolves neo.mjs dev with GraphScene's label layer (#839)

neomjs/neo#20288 merged the label layer and flyTo this branch draws through.
Only package-lock.json moves (neo.mjs 3ae8e2aece -> 8abfcdc6ca); package.json
keeps github:neomjs/neo#dev, so CI's locked-deps unit run sees the engine the
Observatory now needs."
- 2026-11-05T12:01:22Z @neo-opus-vega referenced in commit `d0e1866` - "feat(agentos): the Observatory's wheel zooms in to single nodes (#839)

The operator's read on 11-05: zooming out is fine, zooming in is not enough.
A ~100k-node read draws each node at nodeSizes.nodeMin (0.4 px fitted), so
the engine's floor of half the fitted distance stopped under a pixel. The
Observatory's zoom.min of 0.03 brings a node to ~13 px; the way out keeps
the engine's 3.75, and a well's flight still lands at half the fitted
distance."
- 2026-11-05T12:07:04Z @neo-opus-vega referenced in commit `fb4708e` - "feat(agentos): the Observatory's wheel comes 10x closer, its near level a band (#839)

The operator's read on 11-05: zooming out is fine, zooming in is not enough.
Clio's zoom-ladder design read (#312, 10-16) named the cheap first step:
zoom.min 0.5 -> 0.1 and lod.near 0.55 -> 0.35. A ~100k-node read's floor node
(nodeMin 0.4 px fitted) now reaches a pickable 4 px; near spans the street
band instead of the wheel's last sliver; a well's flight still lands framed at
half the fitted distance. Rendering nodes differently up close (size floor,
fovea labels, the crossfade) is the engine's next step, not this change."
- 2026-11-05T12:07:48Z @neo-opus-vega cross-referenced by #20312
### @neo-opus-vega - 2026-11-05T12:07:59Z

**Operator design read, 2026-11-05:** *"the max zoom out level for observatory feels fine. zooming in is not enough though. … after zooming in up to a specific level, items get rendered differently. the same could in theory happen for our agent os graph."*

**What changed on the branch** (`vega/839-observatory-well-labels` at fb4708e, rebased onto `dev` with the lock at neo `8abfcdc6`, which carries the merged label layer):
- **The wheel comes 10× closer.** `zoom.min` goes 0.5 → 0.1 and `lod.near` 0.55 → 0.35 in `apps/agentos/canvas/Observatory.mjs`. These are exactly step (a) of Clio's zoom-ladder read on #312 (comment 6222093228). A ~100k-node read's floor node now reaches a pickable 4 px, `near` is a band of the travel instead of its last sliver, and a well's flight still lands framed at half the fit.
- A unit arm pins it: red on the old values (0.8 px), green now. The full unit suite passes 2108/0 and the Observatory goldens are unchanged.
- **Rendering differently up close** (a mark floor, names for the nodes in focus, and a crossfade, the `apps/covid` MapBox pattern) is engine mechanics. It's filed as neomjs/neo#20312, mine, with this pane as its first consumer.

**The PR opens next** with the BEFORE/AFTER table from 6645242511 plus a deepest-zoom frame. The wells' labels, halo and Wells section in those frames still stand without objection; this read addressed the zoom only.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-05T12:09:08Z @neo-opus-vega cross-referenced by PR #877
- 2026-11-05T12:24:34Z @neo-opus-vega referenced in commit `1f56058` - "fix(agentos): the zoom step's rationale reads what the engine draws (#839)

The previous commit claimed the deepest zoom grows a dense read's floor node
to 4 px. The engine draws a point at its own size at the camera's target,
whatever the distance, so the step adds room, not size: the nodes around the
target spread to 10x their fitted spacing and a pick lands on the one meant.
The Observatory's zoom JSDoc and its unit arm now say and pin that; the arm is
red on the old zoom.min (2x against the 9.9x bar). The comment also drops its
ticket reference, and the visual stamp is refreshed for the changed
Observatory.mjs."
- 2026-11-05T12:45:22Z @neo-opus-vega cross-referenced by #878
- 2026-11-05T12:54:49Z @tobiu referenced in commit `5f61586` - "feat(agentos): the Observatory names its wells on the canvas and in a Wells section (#839) (#877)

* feat(agentos): the Observatory names its wells on the canvas and in a Wells section (#839)

The wire scene carries the wells (hub index, size, name). The renderer labels
them largest first, 11-15 px by size, in the skin's ink over its ground. Halo
dust takes lineSoft, and well nodes are sized from the wells' own count, so a
well reads as a body of points. A Wells section between Team and Nodes lists
them, "full" at the strategic cap. A row turns the camera to its hub through
ObservatoryCanvas.flyTo; no node moves and the selection stays. readStats
names each drawn label's node by id.

Needs the GraphScene label layer and flyTo of neomjs/neo#20288.

* build(agentos): the lock resolves neo.mjs dev with GraphScene's label layer (#839)

neomjs/neo#20288 merged the label layer and flyTo this branch draws through.
Only package-lock.json moves (neo.mjs 3ae8e2aece -> 8abfcdc6ca); package.json
keeps github:neomjs/neo#dev, so CI's locked-deps unit run sees the engine the
Observatory now needs.

* feat(agentos): the Observatory's wheel comes 10x closer, its near level a band (#839)

The operator's read on 11-05: zooming out is fine, zooming in is not enough.
Clio's zoom-ladder design read (#312, 10-16) named the cheap first step:
zoom.min 0.5 -> 0.1 and lod.near 0.55 -> 0.35. A ~100k-node read's floor node
(nodeMin 0.4 px fitted) now reaches a pickable 4 px; near spans the street
band instead of the wheel's last sliver; a well's flight still lands framed at
half the fitted distance. Rendering nodes differently up close (size floor,
fovea labels, the crossfade) is the engine's next step, not this change.

* fix(agentos): the zoom step's rationale reads what the engine draws (#839)

The previous commit claimed the deepest zoom grows a dense read's floor node
to 4 px. The engine draws a point at its own size at the camera's target,
whatever the distance, so the step adds room, not size: the nodes around the
target spread to 10x their fitted spacing and a pick lands on the one meant.
The Observatory's zoom JSDoc and its unit arm now say and pin that; the arm is
red on the old zoom.min (2x against the 9.9x bar). The comment also drops its
ticket reference, and the visual stamp is refreshed for the changed
Observatory.mjs."
- 2026-11-05T12:54:50Z @tobiu closed this issue

