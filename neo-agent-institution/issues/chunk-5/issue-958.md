---
id: 958
title: The Observatory's node list finds a node by name or kind
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-13T19:25:30Z'
updatedAt: '2026-11-13T19:25:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/958'
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
milestone: FM v1
---
# The Observatory's node list finds a node by name or kind

## Context

Row 3 of FM v1 was walked on the installed candidate on 2026-11-13 ([receipt](https://github.com/neomjs/neo-agent-institution/issues/485#issuecomment-6862535972)). Its check 7 asks the operator to select a session and open it (#333 AC-7). He could not get to one:

> i can not see session or memory nodes. no clue how to activate this, so i can certainly not check if that links anywhere.

## The Problem

The list shows 500 rows of the scene's 141,255 nodes, in one of two orders: changed first, or most related first. It has no search and no filter by kind, and no View control (Wells, Roadmap, Hubs, Attention, Route, Messages, Outside wells) brings up a kind. A node outside the first 500 of both orders can be met only by clicking its few pixels on the canvas.

Sessions are such nodes. The scene holds them: `neomjs/neo#summary_96e284fc-e256-4ab7-9745-0eadea5b1a5c` (kind `SESSION_SUMMARY`) is one, and the panel offers "Open in Memories" once it is selected. None was among the listed rows. So no viewer can walk #333 AC-7's session case, and row 3 cannot pass.

## The Architectural Reality

- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs`: `listBudget = 500` (`:245`); the pane orders the scene's nodes with `ObservatoryNodeList.orderOf(scene, nodeOrder, counts)`, cuts at the budget and lands the rows as records of the node store (`:637`–`:644`), then hands the head `{listed, total}`.
- `ObservatoryNodesHeadContainer.mjs`: the head names the counts and the order, and holds the order toggle.
- The scene from `ObservatorySceneLayout#fromGraphScene` holds every node's `label` and `kind` in the App Worker, so a match needs no new read.
- `apps/agentos/util/GraphNodeSource.mjs#sourceOf` decides what Open does per kind; `SESSION` and `SESSION_SUMMARY` with a uuid open the Memories drill.
- Design authority: the list's JSDoc, "Every node of the bounded read is a row" (`ObservatoryNodeList.mjs:6`), which the budget leaves untrue for a large read, and #333 AC-7, which needs a session selected.

## The Fix

A filter in the Nodes head: a text field that matches a node's label and id, and a choice of kind. The list then shows the first 500 matches in the current order, and the head names the matches beside the listed rows. An empty filter gives today's list.

The filter runs over the scene's nodes, where `orderOf` already runs, and the rows stay records of the node store. Arrow Down from the field enters the rows (#957).

## Acceptance Criteria

- [ ] AC-1 Design read before the PR opens: the field, the kind choice and the count line, approved by the design seat or the operator on #312 or #505.
- [ ] AC-2 A unit arm on the projection: a label substring and a kind each narrow the result, together they intersect, the order holds among the matches, and an empty filter equals today's result (red on `dev` first).
- [ ] AC-3 A unit arm on a scene of 150,000 nodes: a filter answers inside a budget the PR measures and names, and it runs outside the keystroke so typing does not stall.
- [ ] AC-4 An NL/e2e arm on the fixture plane: typing a session's title lists it, and selecting it offers "Open in Memories".
- [ ] AC-5 The head names matches and listed rows. No match reads as a state with its way out.
- [ ] AC-6 BEFORE and AFTER screenshots in the PR body.
- [ ] AC-7 (post-merge, installed) The operator finds a session by its title and opens it in Memories, which is #333 AC-7's session case; the receipt goes on #485.

## Out of Scope

- A Brain-side search over nodes outside the scene read.
- Showing a kind on the canvas (a kind lens).
- Raising the list's budget.

## Avoided Traps

- **Raising the budget to all nodes.** A list of 141,255 rows is a box nobody scrolls, and #509 removed the boxes.
- **A store filter over the landed rows.** It searches only what is already listed, the 500 rows that leave the session out.

## Related

Parent: #312 · #957 (the rows take the keyboard) · #333 and #509 (closed) · #485 (the walk).

unowned-rationale: AC-1's design read comes first; after it the leaf is ready for a peer or a contributor, and row 3's steward reviews.

Live latest-open + closed sweep: latest 20 open issues read at 2026-11-13T18:46Z and the newest six again at 19:24Z, plus state-all searches (`Observatory session`, `Observatory selected node`; Out of Scope and Avoided Traps read on #333 and #509); no equivalent, no prior exclusion. Backlog: 60 open (red); the owning epic is #312 and this is its leaf.
A2A sweep: the last 30 messages, no claim on the Observatory.
MC sweep: "cannot find a session or memory node in the Observatory, 500 of nodes listed, no search, no filter by kind", 5 results, none on this surface; no prior decision.
Own-assignment sweep: 4 open (#732, #651, #649, #485), none overlapping.

Origin Session ID: f6cff01d-943c-47c0-a030-ceaa02bae7ef
Retrieval Hint: "Observatory node list filter by name or kind, session not reachable"

## Timeline

- 2026-11-13T19:25:32Z @neo-opus-vega added the `enhancement` label
- 2026-11-13T19:25:32Z @neo-opus-vega added the `agent-os` label
- 2026-11-13T19:25:32Z @neo-opus-vega added the `ai` label
- 2026-11-13T19:25:33Z @neo-opus-vega added the `design` label
- 2026-11-13T19:25:41Z @neo-opus-vega added parent issue #312
- 2026-11-13T19:25:44Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-11-13T19:26:04Z @neo-opus-vega cross-referenced by #485
- 2026-11-13T19:26:41Z @neo-opus-vega cross-referenced by #312

