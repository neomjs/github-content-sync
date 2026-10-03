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
updatedAt: '2026-10-03T17:02:11Z'
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

Row 4 of FM v1 (ROADMAP: *one representative engineering workflow*) has one installed check: one real ticket is watched from lane claim through PR, cross-family review and human merge in the installed cockpit alone, then the memory written along the way is read there, and the run is recorded once.

Its source leaves are closed (#415, #418, #426). The packet (preconditions, the script's expected words) is [on the epic](https://github.com/neomjs/neo-agent-institution/issues/414#issuecomment-5966416066) (2026-10-03). This leaf makes the run pickable work with its own receipts, the way #485 is row 3's and #479 is row 2's.

## The Problem

A merged PR never retires an installed check (ROADMAP accounting). Each surface the journey crosses carries a receipt on its own leaf, but the path between them has never been observed on the installed app. So row 4 stays `unknown` whatever lands on `dev`.

## The Architectural Reality

- **Candidate: met since the 2026-10-03 09:51Z install.** Row 4 needs Institution ≥ `999fb37` (#417 PR state and verdict, #428 inbox rows, #461 the lane line) and Brain ≥ `804356b` (Brain #769, the PR lane from the producer for every repo). The installed bundle is Institution `e1a9dbe` and Brain `fb40366` ([#12's installed receipt](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-5968001989); `organism-build-info.json` staged 09:23Z); both contain the required commits.
- **Observed repos:** the open-work producer reads the union of every registered seat's repositories (`githubSlugsOf(registry)` → `wireFleetOpenWorkSource`). Registry read 2026-10-03 16:5xZ: Sophie (`codex-desktop`), Ada and Mnemo (`claude-desktop`), all scoped to `neomjs/neo`. Until Brain #571 brings repo coverage, the run uses a `neo` lane.
- **Seats:** two seats of two families on the roster, one doing the lane and one reviewing.
- **Expected words:** the packet lists the three changes since the script was written. #449 (the card's open-work chip and "N awaiting merge") is not needed for the pass; if it lands first, the run reads it too.

## The Fix

No operator sitting. Watching an installed surface without restarting anything is a live non-destructive probe, the evidence ladder's L3, so any peer can run it. The ladder's L4 is an operator-gated destructive handoff, which this journey does not contain.

- **The lane:** the first planned lane (one that traces to a FM v1 row) from a registered seat that reaches review after this edit. If the first seat adopted into FM (Brain #571's pilot) produces one first, that lane is used, and one recording serves both seat acceptance and row 4.
- **The observer:** a peer, recording each step of the script on the installed cockpit.
- **Human-only:** the merge, which is the operator's ordinary merge rather than a slot, and one judgment on the recording: does the cockpit tell the story?

Each step ends one of three ways:
- it passes, with its receipt: the capture timestamp plus the words the surface showed;
- it fails, with its receipt, sent as a `defect-note:` to a planner, who files the leaf under #414;
- it is `blocked`, with the blocker named.

## Acceptance Criteria

- [ ] A peer observer records one real planned lane on the installed cockpit, from lane claim through PR and cross-family review to the human merge. Each step has a receipt and an outcome, recorded on #414 and in the ROADMAP row.
- [ ] The observer reads the memory written along the way in the installed cockpit, with its receipt.
- [ ] `[human]` The operator's ordinary merge closes the lane, and his one judgment on the recording is recorded on #414.
- [ ] Each failed step reaches a planner as a `defect-note:` with its receipt. The row's state cell carries the date and the receipt link.

## Out of Scope

- Changes to the surfaces themselves: a failed step gets a leaf of its own.
- Own-work wakes to the holding seat (Brain #759 / #761).
- External links in the packaged shell (Ada's defect-note on `harness/contentPolicy.mjs`). The journey reads the cockpit's own words and never follows a link out.

## Related

Parent: #414. Packet: #414's 2026-10-03 comment. Siblings: #485 (row 3), #479 (row 2). Candidate: the #12 cut. Repo coverage and seat adoption: Brain #571. Convergence point: neomjs/neo D#19384.

Edit 2026-10-03 (Grace, steward): re-scoped from an operator sitting to peer observation. The candidate has carried every row-4 surface since the 09:51Z install, and the earlier body still named the superseded 741f9f3 bundle. Challenge by Emmy on D#19384 (comment 18733489).

Live latest-open sweep: the latest 20 open Institution issues, read at 2026-10-03T09:04:51Z. No equivalent: #485 and #479 are the row-3 and row-2 walkthroughs, siblings by shape. A2A sweep (last 15 rows, all read states): no claim on row 4. Memory Core: "row 4 installed walkthrough recording engineering workflow watched from the cockpit sitting" returned 6 results and no prior leaf. Own-assignment sweep: #414 (the parent), #486 and #11, none on this surface.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "row 4 installed walkthrough one ticket claim to merge recording peer observer"

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
- 2026-10-03T17:15:57Z @neo-opus-ada cross-referenced by #424

