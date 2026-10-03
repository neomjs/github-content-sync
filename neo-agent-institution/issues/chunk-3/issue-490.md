---
id: 490
title: 'Row 4''s installed walkthrough: one ticket watched from claim to merge'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-03T09:05:09Z'
updatedAt: '2026-10-03T09:05:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/490'
author: neo-opus-grace
commentsCount: 0
parentIssue: 414
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
# Row 4's installed walkthrough: one ticket watched from claim to merge

## Context

Row 4 of FM v1 (ROADMAP: *one representative engineering workflow*) has one installed check. The operator watches one real ticket go from lane claim through PR, cross-family review and human merge in the installed cockpit alone. Then they read the memory written along the way, and the run is recorded once.

Its source leaves are closed (#415, #418, #426). The sitting packet (preconditions, the script's expected words and the operator's calls) is [on the epic](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-5966416066) (2026-10-03). This leaf makes that sitting pickable work with its own receipts, the way #485 is row 3's and #479 is row 2's.

## The Problem

A merged PR never retires an installed check (ROADMAP accounting). Each surface the journey crosses carries a receipt on its own leaf, but the path between them has never been observed on the installed app. So row 4 stays `unknown` whatever lands on `dev`.

## The Architectural Reality

- **Candidate:** the installed Neo Harness from the next #12 cut. It must carry:
  - Institution `dev` ≥ `999fb37`, for #417 (PR state and verdict), #428 (inbox rows) and #461 (the lane line);
  - Brain ≥ `804356b`, for Brain #769 (the PR lane from the producer, for every repo).

  The receipt is `organism-build-info.json`. The bundle installed on 2026-10-03 was staged on Brain `741f9f3`, so it carries none of them.
- **Observed repos:** the open-work producer reads the union of every registered seat's repositories (`githubSlugsOf(registry)` → `wireFleetOpenWorkSource`). A Brain or Institution ticket needs its repo added to a seat in the Agent Detail's Repositories pane first. Otherwise the run uses a `neo` ticket.
- **Seats:** two seats of two families on the roster, one doing the lane and one reviewing.
- **Expected words:** the packet lists the three changes since the script was written. #449 (the card's open-work chip and "N awaiting merge") is not needed for the pass. If it lands first, the sitting reads it too.

## The Fix

One bounded operator slot. The operator picks:
- the ticket and its repo;
- the peer and the reviewer.

The operator also makes the recording. Each step of the script ends one of three ways:
- it passes, with its receipt: the recording timestamp plus the words the surface showed;
- it fails, with its receipt and a leaf under #414;
- it is `blocked`, with the blocker named.

The source audit's gaps are closed. Gaps only the sitting can show become leaves after it (the epic's rule).

## Acceptance Criteria

- [ ] `[L4 — operator slot needed]` One real ticket is watched on the installed candidate, in the cockpit alone, from lane claim through PR, cross-family review and human merge. Each step has a receipt and an outcome, recorded on #414 and in the ROADMAP row.
- [ ] `[L4 — operator slot needed]` The memory written along the way is read in the cockpit, with its receipt.
- [ ] Each failed step files a leaf under #414 with its receipt. The row's state cell carries the date and the receipt link.

## Out of Scope

- Changes to the surfaces themselves: a failed step gets a leaf of its own.
- Own-work wakes to the holding seat (Brain #759 / #761).
- External links in the packaged shell (Ada's defect-note on `harness/contentPolicy.mjs`). The journey reads the cockpit's own words and never follows a link out.

## Related

Parent: #414. Packet: #414's 2026-10-03 comment. Siblings: #485 (row 3), #479 (row 2). Candidate: the #12 cut. Repo coverage for enrolled seats: Brain #571.

Live latest-open sweep: the latest 20 open Institution issues, read at 2026-10-03T09:04:51Z. No equivalent: #485 and #479 are the row-3 and row-2 walkthroughs, siblings by shape. A2A sweep (last 15 rows, all read states): no claim on row 4. Memory Core: "row 4 installed walkthrough recording engineering workflow watched from the cockpit sitting" returned 6 results and no prior leaf. Own-assignment sweep: #414 (the parent), #486 and #11, none on this surface.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "row 4 installed walkthrough one ticket claim to merge recording"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-03T09:05:09Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-03T09:05:11Z @neo-opus-grace added the `enhancement` label
- 2026-10-03T09:05:11Z @neo-opus-grace added the `agent-os` label
- 2026-10-03T09:05:12Z @neo-opus-grace added the `ai` label
- 2026-10-03T09:05:12Z @neo-opus-grace added the `testing` label
- 2026-10-03T09:05:16Z @neo-opus-grace added parent issue #414
- 2026-10-03T09:05:16Z @neo-opus-grace added this to the **FM v1** milestone
- 2026-10-03T09:29:13Z @neo-opus-grace cross-referenced by PR #482
- 2026-10-03T09:53:03Z @neo-opus-vega cross-referenced by #495
- 2026-10-03T10:56:50Z @neo-opus-grace cross-referenced by #498

