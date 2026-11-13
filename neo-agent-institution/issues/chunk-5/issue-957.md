---
id: 957
title: The Observatory shows a list selection's details and takes the keyboard
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-13T19:24:45Z'
updatedAt: '2026-11-13T19:25:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/957'
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
# The Observatory shows a list selection's details and takes the keyboard

## Context

Row 3 of FM v1 was walked on the installed candidate on 2026-11-13 ([receipt](https://github.com/neomjs/neo-agent-institution/issues/485#issuecomment-6862535972)). The operator selected a pull request from the Observatory's node list and could not act on it:

> so, i clicked on it. there is no action => hitting space or enter does nothing. at the very bottom there is a collapsed "selected node" item. terrible UX.

On the keyboard:

> after a node is selected, i need to click on the rather hidden "clear" button to even be able to show other nodes. then i can focus "nodes". from there, no arrow keynav works. so i would need to click on a node inside the list first, to even enable keynav.

And when told that one open section at a time is a recorded decision:

> "One section open at a time" does not necessarily mean "display the details somewhere else, collapsed and hard to find".

## The Problem

Three gaps lie on one path, from seeing a row to acting on it.

1. A selection made in the open node list leaves its details and actions collapsed in the Selected node head at the bottom of the panel.
2. Enter and Space on a selected row do nothing.
3. No key moves focus from the NODES head into the rows. The arrow keys work once a row has focus, and a row gets focus only by a click.

Escape drops a selection, and nothing on screen says so. The visible way back is the Clear action he called hidden.

## The Architectural Reality

- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs`: `afterSetSelectedId` (`:531`) opens the Selected section on a selection unless the node list is open (`:537`, from PR #528). `syncSections` (`:885`) opens one section to the panel's height. `keys: {Escape: 'onEscape'}` (`:130`).
- `ObservatoryNodeList.mjs` extends `Neo.list.Base` with `selectOnFocus: true`. The engine gives each row `tabIndex -1` and navigates through its Navigator addon. The section heads are buttons (`observatory-nodes-title`, `selected-section-head`).
- `ObservatorySelectionContainer.mjs` holds the actions (`selected-open`, `selected-copy`, `selected-clear`).
- Design authority: #509 AC-3, "selecting a node expands Selected with the full title and relations", and the list's own JSDoc, "the path to the scene and its selection that needs no canvas" (`ObservatoryNodeList.mjs:5`). The exception at `:537` contradicts the first for the list; the missing entry contradicts the second for the keyboard. #509 AC-2 (expanding Nodes collapses the other sections) is not called wrong here.

## The Fix

The outcome is fixed; its shape is the design read's (AC-1).

- A row's selection, by click or by key, leaves its details and its actions in view, with no second target to find. Candidates: the Selected section opens, and Escape or a back control returns to the list at the same row; or the selected row expands in place; or the two sections share the height while a list selection exists.
- Enter or Space on a row moves focus to the selection's first action.
- From the NODES head, Arrow Down puts focus on the selected row, else on the first row. Arrow Up on the first row returns to the head.
- The way back is visible where the selection shows.

## Acceptance Criteria

- [ ] AC-1 Design read before the PR opens: a frame of a list selection with its details and actions, approved by the design seat or the operator on #312 or #505. It decides the shape among the candidates.
- [ ] AC-2 An NL/e2e arm on the fixture plane, with real key events from the NODES head: Arrow Down focuses and selects a row; Enter moves focus to the selection's first action; Escape returns focus to that row with the list at the same scroll position (red on `dev` first).
- [ ] AC-3 A unit arm: a selection that comes from the node list leaves the selection's details visible.
- [ ] AC-4 A mouse click on a row needs no second click to read the details and to reach Open.
- [ ] AC-5 BEFORE and AFTER screenshots in the PR body.
- [ ] AC-6 (post-merge, installed) The operator selects a row and opens its source without hunting, by mouse and from the keyboard; the receipt goes on #485.

## Out of Scope

- The look of the actions and of a pressed control: design questions on #505 ([comment](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6862614446)).
- Finding a node that is not among the listed rows: its own leaf.
- The accordion itself (#509 AC-2).

## Avoided Traps

- **Deleting the `:537` exception alone.** Every arrow key would then collapse the list under the viewer, which is why the exception exists. The shape has to keep keyboard browsing.
- **A modal for the node.** #509's avoided trap stands.

## Related

Parent: #312 · #485 (the walk) · #509, #375 and #333 (closed; their criteria are quoted above) · #505 (the design questions).

unowned-rationale: AC-1's design read comes first; after it the leaf is ready for a peer or a contributor, and row 3's steward reviews.

Live latest-open + closed sweep: latest 20 open issues read at 2026-11-13T18:46Z and the newest six again at 19:24Z, plus state-all searches (`Observatory keyboard`, `Observatory selected node`; Out of Scope and Avoided Traps read on #509, #375, #333 and #878); no equivalent, no prior exclusion. Backlog: 60 open (red); the owning epic is #312 and this is its leaf.
A2A sweep: the last 30 messages, no claim on the Observatory.
MC sweep: "Observatory node list cannot be reached by keyboard, arrow keys only after a click, Enter on a row does nothing", 5 results, all from today's walk; no prior decision.
Own-assignment sweep: 4 open (#732, #651, #649, #485), none overlapping.

Origin Session ID: f6cff01d-943c-47c0-a030-ceaa02bae7ef
Retrieval Hint: "Observatory list selection details collapsed keyboard entry Enter row"

## Timeline

- 2026-11-13T19:24:46Z @neo-opus-vega added the `bug` label
- 2026-11-13T19:24:46Z @neo-opus-vega added the `agent-os` label
- 2026-11-13T19:24:46Z @neo-opus-vega added the `ai` label
- 2026-11-13T19:24:47Z @neo-opus-vega added the `design` label
- 2026-11-13T19:25:31Z @neo-opus-vega cross-referenced by #958
- 2026-11-13T19:25:39Z @neo-opus-vega added parent issue #312
- 2026-11-13T19:25:42Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-11-13T19:26:04Z @neo-opus-vega cross-referenced by #485
- 2026-11-13T19:26:41Z @neo-opus-vega cross-referenced by #312

