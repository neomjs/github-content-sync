---
id: 916
title: 'Open-work coverage leaves out benched seats, never their open rows'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-07T11:54:31Z'
updatedAt: '2026-10-07T13:45:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/916'
author: neo-opus-vega
commentsCount: 0
parentIssue: 875
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-07T13:45:32Z'
---
# Open-work coverage leaves out benched seats, never their open rows

## Context

This is sub 2 of #875, proposed in its [closeout review](https://github.com/neomjs/neo-agent-brain/issues/875#issuecomment-6036705718); the operator authorized filing on 2026-10-07. The specimen is Sophie's installed observation ([6025830129](https://github.com/neomjs/neo-agent-brain/issues/875#issuecomment-6025830129)). The Activity feed reported `open-work producer coverage partial` and named failed GitHub reads for `@neo-preview`, whose participation is `operator_benched`, and `@neo-fable-clio`, which is active but dark.

## The Problem

`seatReaders` (`ai/services/fleet/wireFleetOpenWorkSource.mjs`) selects every registry definition that has a GitHub username and a GitHub forge. It never consults participation, so a seat the operator benched still counts toward the coverage the feed must reach. One failed read from a benched seat marks the whole feed partial.

## The Architectural Reality

- Participation is a fact on the identity node. `ai/graph/agentIdentityParticipation.mjs` (`readAgentIdentityNodes`, `participationStatusOf`, `participationByIdentity`) is the shared read; the heartbeat and issue focus already use it.
- `openWorkReducer.mjs` (~L262) can drop retained rows that are absent from a complete observation.

## The Fix

- Required coverage excludes seats whose identity node reads `operator_benched`. Unknown participation stays required, and dark or offline is not benched.
- A benched seat's existing PR and review rows are retained. Leaving a seat out of required coverage must never let the reducer read a complete observation as proof that the seat's rows are gone, because others can still review or merge that work.

## Acceptance Criteria

- [ ] AC-1: a benched seat's failed read no longer marks coverage partial (unit).
- [ ] AC-2, control: an active seat's failed read still marks coverage partial, dark or not (unit).
- [ ] AC-3: unknown participation counts as required (unit).
- [ ] AC-4: a benched seat's retained open-work rows survive a complete observation (unit).

## Post-Merge Validation

- [ ] On the next installed candidate, the Activity feed reads complete while Preview is benched (owner: neomjs/neo-agent-institution#12's walk).

## Out of Scope

- The participation writer (#28).
- Token replacement (#815).
- The roster boot seed: #875's sub 1, a separate leaf.

## Related

#875 (parent) · #880 / #905 (the shared participation read) · neomjs/neo-agent-institution#12

Live latest-open, A2A, MC and own-assignment sweeps were run immediately before filing; none found an equivalent.

Origin Session ID: 7dcf11bd-1a91-43b6-affd-6b4dde2c4088


## Timeline

- 2026-10-07T11:54:33Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-07T11:54:33Z @neo-opus-vega added the `bug` label
- 2026-10-07T11:54:33Z @neo-opus-vega added the `ai` label
- 2026-10-07T11:54:35Z @neo-opus-vega added parent issue #875
- 2026-10-07T12:32:10Z @neo-opus-vega cross-referenced by PR #917
- 2026-10-07T13:16:32Z @neo-opus-vega referenced in commit `90af4f7` - "fix(fleet): a cut review-request list keeps a missed seat's row and opens nothing it may hide (#916)

readBy answered from the visible requested[] alone. When a PR's review requests continue past their first page, a missed benched seat may be one of the unseen requests, so a cut list now counts as readable by any missed seat: the row carries through complete reads, and a cut row first seen after the miss is not counted as opened. A complete list that names no missed seat still leaves, and a participating author's new PR still opens."
- 2026-10-07T13:17:29Z @neo-opus-vega referenced in commit `989ca34` - "test(fleet): the cut-list control's producer options align as the block check reads them (#916)"
- 2026-10-07T13:45:31Z @tobiu referenced in commit `359527b` - "feat(fleet): a benched seat owes open work no coverage, and its rows carry while it misses (#916) (#917)

* feat(fleet): a benched seat owes open work no coverage, and its rows carry while it misses (#916)

The open-work readers take each seat's participation from the roster's
presence read. A seat whose identity node reads operator_benched is still
read when its PAT works, but a missed read no longer makes coverage
partial or names it in the reason. The reducer carries the rows a missed
seat could have read through otherwise complete reads, and a row first
seen after that miss is no `opened`. Unknown participation stays owed;
dark is not benched.

* fix(fleet): a cut review-request list keeps a missed seat's row and opens nothing it may hide (#916)

readBy answered from the visible requested[] alone. When a PR's review requests continue past their first page, a missed benched seat may be one of the unseen requests, so a cut list now counts as readable by any missed seat: the row carries through complete reads, and a cut row first seen after the miss is not counted as opened. A complete list that names no missed seat still leaves, and a participating author's new PR still opens.

* test(fleet): the cut-list control's producer options align as the block check reads them (#916)"
- 2026-10-07T13:45:32Z @tobiu closed this issue

