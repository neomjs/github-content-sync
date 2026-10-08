---
id: 602
title: Loading older Mailbox rows resets the reading position
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - grid
assignees: []
createdAt: '2026-10-08T04:22:44Z'
updatedAt: '2026-10-08T04:22:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/602'
author: neo-gpt-sophie
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
# Loading older Mailbox rows resets the reading position

## Context

The operator reported that scrolling the inbox loads older messages, then sends the view back to the top. An independent installed pass reproduced it: 100 rows at `scrollTop=7194`, start index 85 became 150 rows at `scrollTop=0`, start index 0 after the next 50-row page arrived at offset 100. [Installed receipt](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-6036908714).

That receipt used Institution `85d5282` / Engine `82bc6158`. On October 8 the installed candidate is Institution `fd958fb` with the same Engine pin. A read-only live method inspection confirms its `Grid.applyBags()` still assigns `store.data = bags`. A fresh native gesture attempt timed out in the UI tool before observation; this ticket does not claim a second installed reproduction.

Design authority: operator, October 7: “as soon as they load, the view uses the scroll state, and scrolls all the way to the top (which is bad).” Reading older messages must remain possible without repeatedly finding the previous position.

## The Problem

A page continuation is presented to the grid as a whole replacement. The operator loses the reading position every time the loaded corpus grows. This was captured as an admitted defect note; the live tracker has no dedicated repair leaf.

## The Architectural Reality

At Institution `68bd58cb1ee59d49b28360f69d5cba7831944356`:

- `apps/agentos/view/fleet/mailbox/Container.mjs#applySnapshot` recognizes `snapshot.page.offset > 0`, concatenates held bags with the next page, then calls the grid's single mutation path.
- `mailbox/Grid.mjs#applyBags` stamps thread facts and replaces `store.data`. Its class documentation deliberately requires coherent thread facts and fresh record identities to avoid stale or double-mounted pooled cells.
- Engine `data.Store.afterSetData` clears and adds; `grid.Body.onStoreLoad` schedules a scroll-to-top dispatch for a mounted body unless the notification is a continuation (`postChunkLoad`). Continuation handling already exists; globally changing ordinary-load behavior is not the repair.
- The current mailbox also exposes selection and a message-detail pane. Preserving scroll must not silently discard that selection or leave stale detail state.

Structure map: `ai:structure-map` is not hosted in the resident Engine checkout (Missing script, exit 1). Placement is existing Institution mailbox view code and its owning tests; no new subsystem or module is proposed.

## The Fix

Preserve the reader's position through the existing mailbox page-continuation projection, starting at `Container.applySnapshot` and `Grid.applyBags`. Use the Store/Grid continuation semantics while retaining the single coherent thread-fact/cell refresh contract. Verify the Engine version actually consumed by Institution; an unconsumed Engine change is not consumer evidence.

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Mailbox page continuation | Operator report and the installed receipt above | Older rows extend the corpus without moving the visible reading anchor | First-page replacement or a different subject retains intentional replacement behavior | Existing mailbox projection JSDoc | Real browser paging, plus thread/selection controls |

Decision Record impact: none.

## Acceptance Criteria

- [ ] Scrolling through at least two additional pages preserves the visible message and its viewport offset across each settled projection; there is no intermediate jump to the first row.
- [ ] A thread spanning a page boundary keeps correct head/count facts and its existing collapsed/expanded state, with no stale or duplicated pooled cells.
- [ ] A selected message and its detail remain coherent while older rows arrive; no read/reply/resolve operation is performed merely by paging.
- [ ] A genuine first-page replacement, subject change or shorter replacement corpus still produces a valid scroll position and clears stale selection where appropriate.
- [ ] Loading remains edge-driven and bounded: no boot drain, duplicate next-page requests, or extra page loop is introduced.
- [ ] A browser regression fails on the current replacement path and passes on the repair, with logical rows, rendered rows and scroll position checked together.

## Post-Merge Validation

The next authorized installed FM candidate repeats older-page scrolling with real operator data and records its product/Engine pins on #12. Source validation alone does not close that installed acceptance.

## Avoided Traps

- Calling a private Store helper from the app or teaching every ordinary grid load to preserve stale scroll.
- Adding rows without updating thread facts or respecting pooled-cell record identity.
- Restoring scroll only after a visible jump, or hiding it with fixed delays.
- Reintroducing the mailbox boot drain while testing multiple pages.

## Out of Scope

Mailbox permission policy, task auto-resolution, docking/popup restoration, and broad view refactoring.

Related: #12, #414, #505, #429, #416. The merged own-message detail work remains unchanged.

unowned-rationale: ready for the next Institution implementation slot; current named source lanes continue. Sophie retains installed FM acceptance coordination.

Discovery: live latest-open Institution issues and all-state recent A2A were checked immediately before creation on 2026-10-08; no matching issue or implementation claim. Historical `scroll` / `mailbox` searches found edge-fetch and boot-drain work, not position retention. KB had no useful match. MC `Fleet Mailbox scroll jumps top pagination load` recovered the original measured defect and its single-projection rationale (session `ccd79763-75f3-4295-9805-04d7171926ff`). Own-assignment sweep: only Accounts #601, a separate drag-admission surface.

Origin Session ID: e6ce4d70-a7ff-454e-996d-e7c25efdf4cf
Retrieval Hint: "Mailbox older page 7194 scrollTop zero applyBags store.data"


## Timeline

- 2026-10-08T04:22:45Z @neo-gpt-sophie added the `bug` label
- 2026-10-08T04:22:46Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-08T04:22:46Z @neo-gpt-sophie added the `ai` label
- 2026-10-08T04:22:46Z @neo-gpt-sophie added the `grid` label
- 2026-10-08T04:23:30Z @neo-gpt-sophie cross-referenced by #12

