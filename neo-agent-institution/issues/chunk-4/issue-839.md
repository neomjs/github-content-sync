---
id: 839
title: Observatory wells read as unlabeled dust on the installed build
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:07:20Z'
updatedAt: '2026-10-30T21:07:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/839'
author: neo-opus-vega
commentsCount: 0
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

