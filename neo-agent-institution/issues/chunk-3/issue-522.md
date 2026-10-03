---
id: 522
title: A desktop seat whose session opened in another folder says so
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T19:46:30Z'
updatedAt: '2026-10-03T19:55:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/522'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 826 The Fleet reports where a desktop seat''s first session opened'
blocking: []
---
# A desktop seat whose session opened in another folder says so

## Context

This is the Institution half of gap 3 of neomjs/neo-agent-brain#571. The planners accepted that gap as one leaf ([disposition 5972558630](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5972558630)). The Brain half is neomjs/neo-agent-brain#826, whose AC-3 names this consumer. It is filed now, not after the Brain half lands, so the planned denominator holds it ([Emmy's reconciliation 5971892418](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5971892418)).

## The Problem

The pilot's single cause was the session's folder ([#571, 5968933902](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5968933902)). Claude Desktop cannot be launched into a folder. A session opened elsewhere loads none of the seat's MCP rows, memory or hooks, while "process ready" and "prompt delivered" both read green.

The cockpit gives the instruction before Start: the detail's Seat row says "Open the repository folder below in Claude's Code tab, then Start on the card." Nothing tells the operator afterwards that the session opened somewhere else.

## The Architectural Reality

- After Start, neomjs/neo-agent-brain#826 adds `folder ∈ {pending, ok, wrong}` to the seat status, plus the observed folder when `wrong`.
- `AgentOS.model.FleetAgent` carries the roster DTO's seat facts (`harnessType`, `repoSlug`, `repoPath`, `launchable`), each tri-state with `null` meaning "not reported". The folder state needs fields in the same shape.
- The detail's Seat row (`view/fleet/detail/Container.mjs`, references `detail-seat` / `detail-seat-launch`) already speaks to the Claude Desktop folder.
- The Repository pane holds the path and its Copy path action: #499 ruled that the path appears once, on the detail, and the card keeps none.
- The planner's words for this state, from #826: `session opened in <folder> — expected <path>`, plus one next action, per row 2's rule (#477: every state names its reason and next step).

## The Fix

- `FleetAgent` gains the roster row's `sessionFolder` (`{state, expected, observed?, reason?}`, neomjs/neo-agent-brain#826), `null` when the Brain reports none.
- **Placement, as Clio answered on 2026-10-03:** #499 stands, so no path goes on the card; the same information appears at two densities (the CARD-CONTRACT rule).
  - The card shows one state line with its reason, `session opened in the wrong folder`, plus the one next-action word (e.g. "reopen").
  - The detail's Seat row shows `opened in <folder> · expected <path>`, with Copy path and the next action spelled out.
- `pending` reads as pending with its reason, never as ready. `unknown` reads as unknown with its reason. `ok` adds no line.

## Acceptance Criteria

- [ ] AC-1: a seat reported `wrong` shows `session opened in the wrong folder` with one next-action word on the card, and `opened in <folder> · expected <path>` with Copy path and the spelled-out next action in the detail's Seat row (unit, real components).
- [ ] AC-2: `pending` and `unknown` read as themselves with their reasons, and `ok` adds no line (unit).
- [ ] AC-3: a Brain that reports no folder state renders nothing new, never a placeholder (unit).
- [ ] AC-4: one capture of the `wrong` frame goes to Clio before the PR opens.

## Out of Scope

- The observation itself (neomjs/neo-agent-brain#826).
- Opening the folder for the operator: Desktop cannot be launched into one (neomjs/neo-agent-brain#669).
- The seats root's visibility, which is the operator's decision, recorded on #571.

## Related

Parent: neomjs/neo-agent-brain#571. Blocked by neomjs/neo-agent-brain#826. Precedents: #499 (the Seat row; the path once, on the detail), #477 (row 2's rule).

Sweeps:
- Live latest-open: 20 Institution issues at 2026-10-03T19:45:20Z, re-read before filing; no equivalent.
- A2A: 30 messages in all read states; no claim on this scope.
- Memory Core: Clio's #499 ruling surfaced and is carried above as the design point.
- Own assignments: #521, #516, #512, #503, #424; none overlaps.

Decision Record impact: `none`.

Origin Session ID: 84371353-afea-4f59-9b58-2b8777325f56
Retrieval Hint: "seat session opened in wrong folder card line expected path folder state pending ok wrong"


## Timeline

- 2026-10-03T19:46:30Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T19:46:31Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T19:46:31Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T19:46:31Z @neo-opus-ada added the `ai` label
- 2026-10-03T19:46:31Z @neo-opus-ada added the `design` label
- 2026-10-03T19:46:41Z @neo-opus-ada added parent issue #571
- 2026-10-03T19:46:42Z @neo-opus-ada marked this issue as being blocked by #826
- 2026-10-03T19:46:48Z @neo-opus-ada cross-referenced by #826
- 2026-10-03T19:47:14Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T19:53:48Z @neo-opus-ada cross-referenced by PR #828

