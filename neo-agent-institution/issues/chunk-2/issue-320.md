---
id: 320
title: 'The Observatory''s heat overlay and team lens: attention and attribution over any geography'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-29T11:48:22Z'
updatedAt: '2026-09-29T14:52:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/320'
author: neo-opus-vega
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
# The Observatory's heat overlay and team lens: attention and attribution over any geography

## Context

Stage C of D#19317 had two overlays and a lens beside its geography. #311 now ships the geography only: W3 wells with the mass cap, the W2 fallback, and the #620 envelope landing. This leaf carries the rest, moved out of #311 so that each lands as its own reviewable PR. The Brain columns this leaf reads:
- B1, neomjs/neo-agent-brain#603 (merged): `lastActivityAt` per kind, with `activitySources` naming each kind's source field.
- B2, neomjs/neo-agent-brain#604 (merged): `authoredBy`, `assignedTo` and `memoryOf`, with the origin carried by the identity.
- B3, neomjs/neo-agent-brain#625 (open): each issue's, PR's and discussion's state. AC-1 needs it, because an item's `updatedAt` cannot say whether it is merged. The graph already stores the state, but the scene never projected it.

## The Problem

- "Work focus areas" (Q2) means what the team touched lately. A bare timestamp does not say which event happened, and a closed or bot-updated item is not attention now (D#19317 STEP_BACK point 4).
- The operator's lens direction is *"we want to see all your AND emmy's nodes"*: one checkbox per peer, the checked peers shown as a union, one colour per identity with a default (OQ-W9).

## The Architectural Reality

- The overlays draw over whatever geography is chosen, and switching one moves no node (D#19317 §5). They are channels on `ObservatorySceneLayout`'s scene and the canvas worker's buffers, beside `positions` and `clusters`.
- The right-hand panel contract (D#19317 §4): team controls at the top, the selected node's details beneath.
- No registry field holds a peer's colour: `metadata.avatarUrl` is the registry's one presentation field, and `setAvatar` its writer. Hues hashed from the whole wheel collide on the roster: `@neo-gpt`, `@neo-preview` and `@tobiu` fall within 7° of each other, and `@neo-gemini-pro` and `@neo-opus-ada` 5° apart.

## The Fix

- **Heat (W5):** an overlay over a named event taxonomy in a stated window. The taxonomy maps each `activitySources` field to an event and decides which events count as attention. Kinds with no source propagate heat from neighbouring work events, and a failed source reads unknown.
- **Team lens:** one checkbox per peer, with the checked peers shown as a union. Labels are authored, assigned and recently changed. There is no "working now" mark (OQ-W8).
- **Peer colours:** the palette default keyed on identity, never on the model: eight hues outside the route's gold, each identity's own place drawn by a hash, and the peers shown together probed to places of their own.
- **The strategic line (AC-6):** today `strategicWellsOf` walks the graph twice, once uncapped to measure what is reached and once with the cap (#321's review, point c). The first pass becomes a reachability-only walk. Its result is what splits the halo into "no well reached" and "over a well's cap".

## Acceptance Criteria

- [ ] AC-1 Heat uses the named event taxonomy: a merged item retires, and a failed source reads "unknown". (Was #311 AC-2.)
- [ ] AC-2 The lens shows the union of the checked peers, an old assignment never reads as current work, and no "working now" mark exists. (Was #311 AC-3.)
- [ ] AC-3 Peer colours come from the palette default, keyed on identity, and the checked peers share no hue until the palette runs out. (Was #311 AC-4; its registry field moved to Out of Scope.)
- [ ] AC-4 Unit arms go red on `dev`, and NL arms read the lens and overlay state.
- [ ] AC-5 (post-merge, installed) The operator's viewer, launched cold from a saved plane, shows the overlays and the lens.
- [ ] AC-6 The line reads the strategic cap: the halo splits "no well reached" from "over a well's cap", so its count stops conflating two facts, and `wellCap` gets its first reader. (From #321's review 5352528441.)
- [ ] AC-7 The pane has one admission source. Either the scene envelope's `admission` replaces the Golden Path leaf's copy for the route overlay, or the scene copy leaves the SHAPE. The choice is pinned by an arm. (From #321's review 5352528441.)
- [ ] AC-8 (post-merge) #311's AC-1 residual: on a plane serving neomjs/neo-agent-brain#611's columns, record the strategic wells' sizes and `wellCap` on this ticket. Once neomjs/neo-agent-brain#625 is merged and deployed, the same measurement also records that the answer carries its `states` and `nodes.state` column (#625's AC-4).

## Out of Scope

- "Working now" and the compact peer briefing (OQ-W8).
- Editing a peer's colour, which belongs to the agent-setup view, and with it the registry field: a stored colour whose only writer is the default its identity already determines adds nothing until the operator can choose one.
- The geography (#311).

## Related

#311 (the geography, split from it) · D#19317 · #312 · neomjs/neo-agent-brain#603 · neomjs/neo-agent-brain#604

Decision Record: NOT_NEEDED (D#19317)

## Signal Ledger
Inherited from #311, which graduated from D#19317 at body `updatedAt 2026-09-28T10:56:25Z`: `claude` AUTHOR_SIGNAL (@neo-opus-vega), `gpt` APPROVED (@neo-gpt).

## Unresolved Dissent
(none)

## Unresolved Liveness
As on #311.

## Discussion Criteria Mapping
- Q2 and W5 → AC-1
- Q4, historical half, and OQ-W7 → AC-2
- OQ-W9 → AC-3

Live latest-open sweep: latest 20 open Institution issues, read 2026-09-29 ~12:00Z, plus an org search for "heat overlay" and "team lens"; only #311 and #312, neither equivalent. A2A: this lane is claimed under #311 and nobody else claims it.

Origin Session ID: db0e34f7-9d0f-4799-a2c2-3a5033f8bc9a

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿






## Timeline

- 2026-09-29T11:48:23Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-29T11:48:23Z @neo-opus-vega added the `enhancement` label
- 2026-09-29T11:48:24Z @neo-opus-vega added the `agent-os` label
- 2026-09-29T11:48:24Z @neo-opus-vega added the `ai` label
- 2026-09-29T11:48:24Z @neo-opus-vega added the `design` label
- 2026-09-29T11:48:56Z @neo-opus-vega cross-referenced by #311
- 2026-09-29T11:59:31Z @neo-opus-vega cross-referenced by PR #321
- 2026-09-29T13:36:28Z @neo-opus-vega cross-referenced by PR #626
- 2026-09-29T14:54:58Z @neo-opus-vega referenced in commit `ac57d37` - "feat(observatory): the line names what a full well refused, and the pane reads one admission (#320)

The strategic walk's uncapped pass becomes a reachability walk (reachOf), and its flags split the halo:
fromGraphScene counts overCap, the nodes an anchor reached that only a full well could have taken, and
the line names them with the well cap. The graph read's copy of the route's admission leaves the
envelope's closed shape: the Golden Path leaf, which alone knows when the route expired, stays the
pane's one admission source."
- 2026-09-29T14:54:58Z @neo-opus-vega referenced in commit `638d9da` - "feat(observatory): the team lens and the heat draw over any geography (#320)

The Team list holds the peers the read attributes nodes to. Checked peers draw their
union in hues of their own, a route's seeds included, the rest fades, and the node list
names what each node is to its peer; an assignment untouched past the window never reads
as current work. The Heat toggle brightens what drew attention within three days by a
named event taxonomy: open work by recency, a merged or closed item retired, memories and
gaps counted, messages and file times not, a kind without a source lent half its hottest
neighbour's heat, and a state or time the read lacks reads unknown. Neither moves a node.

Peer hues come from eight places outside the route's gold, each identity's own place
drawn by a hash and the peers shown together probed to places of their own: hues hashed
from the whole wheel put five of nine roster identities within 5 degrees of another."
- 2026-09-29T15:02:13Z @neo-opus-vega referenced in commit `ddc2332` - "fix(observatory): a long line gives way to the hovered node, and the hint to the line (#320)

The head's line and its hover slot shared one nowrap row whose overflow cut the end, so a
withheld route pushed the gesture hint out of the head, and the lens and heat clauses reach
it with both overlays on. The line keeps its width while the hint shows, and ellipsizes
under a hovered node, which keeps its own."
- 2026-09-29T15:02:49Z @neo-opus-vega cross-referenced by PR #325
- 2026-09-29T15:31:55Z @neo-opus-vega referenced in commit `518ca68` - "feat(observatory): the line names what a full well refused, and the pane reads one admission (#320)

The strategic walk's uncapped pass becomes a reachability walk (reachOf), and its flags split the halo:
fromGraphScene counts overCap, the nodes an anchor reached that only a full well could have taken, and
the line names them with the well cap. The graph read's copy of the route's admission leaves the
envelope's closed shape: the Golden Path leaf, which alone knows when the route expired, stays the
pane's one admission source."
- 2026-09-29T15:31:56Z @neo-opus-vega referenced in commit `529f1c6` - "feat(observatory): the team lens and the heat draw over any geography (#320)

The Team list holds the peers the read attributes nodes to. Checked peers draw their
union in hues of their own, a route's seeds included, the rest fades, and the node list
names what each node is to its peer; an assignment untouched past the window never reads
as current work. The Heat toggle brightens what drew attention within three days by a
named event taxonomy: open work by recency, a merged or closed item retired, memories and
gaps counted, messages and file times not, a kind without a source lent half its hottest
neighbour's heat, and a state or time the read lacks reads unknown. Neither moves a node.

Peer hues come from eight places outside the route's gold, each identity's own place
drawn by a hash and the peers shown together probed to places of their own: hues hashed
from the whole wheel put five of nine roster identities within 5 degrees of another."
- 2026-09-29T15:31:56Z @neo-opus-vega referenced in commit `82d989b` - "fix(observatory): a long line gives way to the hovered node, and the hint to the line (#320)

The head's line and its hover slot shared one nowrap row whose overflow cut the end, so a
withheld route pushed the gesture hint out of the head, and the lens and heat clauses reach
it with both overlays on. The line keeps its width while the hint shows, and ellipsizes
under a hovered node, which keeps its own."
- 2026-09-29T15:33:57Z @neo-opus-vega referenced in commit `fd18512` - "docs(observatory): the heat names how fresh the state it reads is (#320)

A work item's state and time are what the Brain's last ingestion stored, and the read carries no freshness, so an item merged since still heats until the next ingestion. The lifecycle is a work item's, never a task's."

