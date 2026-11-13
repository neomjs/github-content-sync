---
id: 1685
title: 'A strategic anchor outlives its ticket: the flag and its weight only rise'
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-11-13T19:30:28Z'
updatedAt: '2026-11-13T19:30:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1685'
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
# A strategic anchor outlives its ticket: the flag and its weight only rise

## Context

The Fleet Manager's Observatory draws the graph's strategic anchors as its "Roadmap" wells. On the installed candidate (Brain `7b3076c8`), walking FM v1's row 3 on 2026-11-13, the operator read them and said:

> stale content too. this is not a FM but an agent os problem => why show closed or resolved tickets here?

Receipt of the walk: [neomjs/neo-agent-institution#485](https://github.com/neomjs/neo-agent-institution/issues/485#issuecomment-6862535972).

## The Problem

Of the 48 wells in that read, 15 are tickets or pull requests. I read each one's state on GitHub: **all 15 are closed or merged**, between 2026-04-24 and 2026-09-15 (`neomjs/neo` 13624, 15270, 10845, 15316, 10308, 16053, 14976, 17156, 17999, 16456, 12301, 10303, 18743, 10829, 15983). None is open. The other wells are classes, concepts and old sessions ("Engine v13.3.1 Release Session", "v13 Release & Identity Convergence").

So the view that is named after the roadmap shows where the work once was. A reader takes a well for "this matters now".

The cause is in how an anchor is stored. The semantic extraction pass marks a node as a "long-term strategic anchor" and gives it a weight. On every later pass the stored node keeps the flag, and its weight becomes the larger of the old and the new value. Nothing lowers either one, and nothing reads whether the work item behind the node is still open.

## The Architectural Reality

- `ai/services/graph/SemanticGraphExtractor.mjs:451`–`:458`: `gravity_well` is set when the pass says `true` and is never unset; `strategic_weight` is `Math.max(stored, new)`. A search of `ai/` for both names finds no other write.
- `ai/services/memory-core/GraphService.mjs:1530`: `readSceneGraph` hands both out as the scene's `gravityWell` and `strategicWeight` columns (#603). `decayGlobalTopology` (`:1091`) decays edges and does not touch them.
- The consumer, `apps/agentos/util/ObservatorySceneLayout.mjs:321` in `neomjs/neo-agent-institution`: an anchor's pull is `strategicWeight × log(1 + degree)`, and the 48 strongest become wells. Old, well-connected, finished work wins that ranking for good.
- The scene already carries each work item's `state` and `lastActivityAt` (#603), so the fact that an item closed is known to the plane.
- Design authority: the extraction schema's own sentence for the flag, "Is this a long-term strategic anchor from roadmap/boardroom?" (`SemanticGraphExtractor.mjs:990`), and the layout's JSDoc, wells "show where the roadmap's weight lies" (`ObservatorySceneLayout.mjs:12`). A merged pull request from April is neither. No decision record or guide under `learn/` names either field (searched for `gravity well` and `strategic weight`).
- Structure map (`npm run ai:structure-map -- --files --loc`, 2026-11-13): the owner is `ai/services/graph`.

## The Fix

An anchor is a statement about now, so it has to be able to end.

1. **A work item's anchor ends with the item.** When the plane learns that an issue, pull request or discussion is closed or merged, its node loses `gravity_well`. Its weight may stay as history; without the flag it attracts nothing.
2. **Every anchor carries when it was last asserted.** The extraction pass stamps the time each time it names a node an anchor. `readSceneGraph` answers the flag only for an anchor asserted inside a window, so an old session or a concept nobody has named for months lapses without anyone deciding it. The window is one config leaf (read ADR 0019 before adding it).

My recommendation is both, in one change: the first fixes what the operator saw, the second keeps it from coming back through kinds that have no state. If the graph's owners want only one, the first is the smaller.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_graph_scene` → `nodes.gravityWell` | #603; `GraphService#readSceneGraph` | `1` only for a live anchor: not a closed or merged work item, and asserted inside the window | a consumer on an older Brain reads today's values | the method's JSDoc | the unit arms below and the installed receipt |

## Acceptance Criteria

- [ ] AC-1 A unit arm: a node flagged as an anchor whose work item is closed or merged reads `gravityWell: 0` in the scene, and an open one reads `1` (red on `dev` first).
- [ ] AC-2 A unit arm: an anchor asserted outside the window reads `0`, and a pass that names it again brings it back.
- [ ] AC-3 The extraction pass no longer keeps the flag unconditionally: the write at `SemanticGraphExtractor.mjs:451` records the assertion time.
- [ ] AC-4 A measurement before and after on the plane's own graph, in the PR body: how many flagged anchors are closed work items, and how many remain live.
- [ ] AC-5 JSDoc on the changed methods says what makes an anchor live.
- [ ] AC-6 (post-merge, installed) The Observatory's Roadmap wells hold no closed issue and no merged pull request; the receipt goes on neomjs/neo-agent-institution#485.

## Out of Scope

- How the cockpit chooses and names wells among live anchors: an Institution leaf under neomjs/neo-agent-institution#312 if the walk asks for one.
- The Golden Path route's ranking (#1389, closed).
- The non-canonical issue identities (`ISSUE:N`, `ISSUE:#N` beside a canonical twin); two of the 48 wells are one issue under two ids, which neomjs/neo-agent-institution#333 already names as a Brain data defect.

## Avoided Traps

- **Hiding closed wells in the cockpit only.** The flag also feeds every other reader of the scene and of the graph, and the next consumer would show the same stale anchors.
- **Decaying the weight alone.** A flag that never clears leaves a zero-weight anchor an anchor; the ranking changes and the meaning stays wrong.

## Related

#603 (the columns) · #1389 (closed; the route's version of "never fades") · neomjs/neo-agent-institution#312 and #485 (the Observatory and the walk).

unowned-rationale: the choice between the two mechanisms belongs to whoever takes the graph's extraction; the problem and both places are measured here, and row 3's steward reviews the consumer side.

Live latest-open + closed sweep: latest 20 open issues read and state-all searches (`gravity well`, `gravity_well`, `strategic weight decay`, `Roadmap wells closed`, `golden path stale closed`; the Problem, Fix and Out of Scope read on #1389, #603 and #122) at 2026-11-13T19:28:22Z; no equivalent, no prior exclusion. Backlog: 154 open (red); no open ticket or epic owns the anchors' lifetime (#122 owns consumers over the route).
A2A sweep: the last 30 messages, no claim on the graph's anchors.
MC sweep: "Observatory shows closed or resolved tickets as roadmap wells, stale gravity wells, strategic anchors never retire or decay", 6 results, none on this; no prior decision.
Own-assignment sweep: 17 open, none overlapping.

Origin Session ID: f6cff01d-943c-47c0-a030-ceaa02bae7ef
Retrieval Hint: "gravity_well never cleared, strategic_weight Math.max, Roadmap wells closed tickets"

## Timeline

- 2026-11-13T19:30:30Z @neo-opus-vega added the `bug` label
- 2026-11-13T19:30:30Z @neo-opus-vega added the `ai` label
- 2026-11-13T19:30:30Z @neo-opus-vega added the `architecture` label
- 2026-11-13T19:30:30Z @neo-opus-vega added the `agent-os` label

