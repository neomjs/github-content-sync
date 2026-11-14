---
id: 972
title: The Observatory's split leaves an existing selection's row out of view
state: OPEN
labels:
  - bug
  - accessibility
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T16:36:46Z'
updatedAt: '2026-11-14T16:52:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/972'
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
# The Observatory's split leaves an existing selection's row out of view

## Context

Row 3 of FM v1 (#312). #957 built the list selection's split: candidate (c) of its AC-1 design read, with the list above and the selected node below. The split reached `dev` through PR #966 (`f563f0ae`). I reviewed PR #965 at `8e56550` on the lockfile's engine `2f38667e68`, where the PR's own `FleetObservatoryNL.spec.mjs` passed 10 of 10. A throwaway arm then used a 60-node fixture with row 40 selected. It measured three paths where the split shows a selection whose row stays out of view:

| Path | Row 40 focused | Row 40 in view | List `scrollTop` |
|---|---|---|---|
| Control: a click on row 40 | yes | yes | 584 |
| A: the list scrolled to the top, then ArrowDown on the Nodes head | yes | no | 0 |
| B: a canvas selection while Team is open, then ArrowDown on the collapsed Nodes head | yes | no | 0 |
| C: the same selection, then a mouse click on the Nodes head | no | no | 0 |

The PR's arms enter from the head only onto row 0, and a scroll to the top always shows row 0.

## The Problem

On A and B, keyboard focus lands on a row the viewer cannot see. On C, the details show below a list that hides their row. B is the walk most likely to meet it: pick a node on the canvas, then move to the list by key. The first ArrowDown seems to do nothing. The second selects the next row, also out of sight, and only that change scrolls.

## The Architectural Reality

- `ObservatoryNodeList#navigator` sets `scrollIntoView: false`, so `Neo.main.addon.Navigator#navigateTo` focuses a row with `preventScroll: true`, and `ObservatoryNodeList#revealRecord` is the list's only scroller.
- `ObservatoryContainer#afterSetSelectedId` calls `revealRecord` only when the selection changes while Nodes is open. `ObservatoryNodeList#enterFromHead` focuses without revealing, and a split opened over an existing selection (`afterSetOpenSection` → `syncSections`) reveals nothing.
- Each reveal measures and then calls `scrollBy` with a relative delta, so two reveals of one row in flight can add up.
- The query field of #958 enters the list through the same `listEnter`.
- `ObservatoryContainer.mjs` has 993 lines on `dev`, and `buildScripts/checkAppFileSizes.mjs` errors above 1,000.
- Design authority: "The selected row is fully in view with one row of context below it." (condition 1 of the AC-1 design read, #957 comment 6870969619), and the list's JSDoc on `navigator`: "{@link #revealRecord} owns row scrolling so a split selection keeps one row of context below the active row".

## The Fix

- `ObservatoryContainer#syncSections` reveals the selection when its caller asks: a selection change under the split, and Nodes opening over a selection. It waits until Selected and its relations have settled. This replaces the reveal in `afterSetSelectedId`. A new read does not ask (Avoided Traps).
- `ObservatoryNodeList#enterFromHead` reveals the row it focuses.
- `ObservatoryNodeList#revealRecord` runs one reveal at a time, and each measures after the previous one has scrolled.
- Escape from Selected stays as built: focus returns to the row at the held `scrollTop` (#957 AC-2).

## Acceptance Criteria

- [ ] AC-1 An NL arm on the fixture plane, red on `dev` first. The selected row starts outside the list's viewport, either because (A) the list was scrolled away or (B) the selection was made on the canvas while Team was open. ArrowDown on the Nodes head leaves the row focused and fully in view, with the next row. (C) A click on the Nodes head over that selection leaves the row fully in view.
- [ ] AC-2 The existing arms of `FleetObservatoryNL.spec.mjs` stay green, including Escape from Selected at the held `scrollTop`.
- [ ] AC-3 The Observatory goldens pass unchanged at the pinned engine, except `observatory-pane-widened`, which fails on `dev` as well (a canvas-only drift, defect-note `e3cd9088`). The visual-baseline stamp is rewritten for the touched inputs.
- [ ] AC-4 `ObservatoryContainer.mjs` stays at or under 1,000 lines (`checkAppFileSizes`).
- [ ] AC-5 (post-merge, installed) During #957's AC-6 walk on #485: select on the canvas, then press ArrowDown on the Nodes head, and the row is in view.

## Out of Scope

- A reveal when Escape returns from Selected: #957 AC-2 holds the scroll position there.
- How the query field of #958 filters: only its ArrowDown entry shares `listEnter`.
- Refreshing `observatory-pane-widened`, which fails on `dev` without this change (defect-note `e3cd9088`).

## Avoided Traps

- Turning the Navigator's `scrollIntoView` back on: its smooth `block: 'nearest'` keeps no context row, and it would move `scrollTop` when Escape returns focus.
- A reveal on every focus move: rejected for the same reason.
- A reveal on every new read: a read that lands while the viewer is reading other rows would pull the list back to the selection.
- A new method on the pane: it is seven lines under its ceiling, and the reveal's mechanics belong to the list.

## Related

Parent: #312 · #957 (closed; the split) · PR #965 (closed) · PR #966 (merged the work) · #958 · #485 (the walk).

Ownership: self-assigned as row 3's steward, in flight now, so no `good first issue`. Maryam, the split's author, reviews.

Live latest-open + closed sweep: latest 20 open read at 2026-11-14T16:35Z, plus a closed search (`Observatory selected row scroll view`; Out of Scope and Avoided Traps read on #957). No equivalent, no prior exclusion. Backlog: 59 open (red); the owning epic is #312 and this is its leaf.
A2A sweep: the last 30 messages, no claim on the Observatory's reveal.
MC sweep: "Observatory node list selected row out of view, keyboard focus lands on a hidden row, list does not scroll to the selection", 5 results, no prior decision.
Own-assignment sweep: 4 open (#732, #651, #649, #485). #485 holds this leaf's post-merge receipt; none overlap.

Origin Session ID: 4c2e4f5b-144f-4493-8fe0-114a5597d003
Retrieval Hint: "Observatory split selected row out of view head entry reveal"

## Timeline

- 2026-11-14T16:36:46Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T16:36:47Z @neo-opus-vega added the `bug` label
- 2026-11-14T16:36:48Z @neo-opus-vega added the `accessibility` label
- 2026-11-14T16:36:48Z @neo-opus-vega added the `agent-os` label
- 2026-11-14T16:36:48Z @neo-opus-vega added the `ai` label
- 2026-11-14T16:36:48Z @neo-opus-vega added the `design` label
- 2026-11-14T16:36:51Z @neo-opus-vega added parent issue #312
- 2026-11-14T16:36:57Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-11-14T16:53:33Z @neo-opus-vega cross-referenced by PR #973

