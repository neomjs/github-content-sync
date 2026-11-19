---
id: 1851
title: A Round-2 comment that keeps a change request routes route-review
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-19T12:04:50Z'
updatedAt: '2026-11-19T13:28:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1851'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-19T13:28:54Z'
---
# A Round-2 comment that keeps a change request routes route-review

## Context

The Fleet tells a PR's author to request a reviewer when the reviewer has just kept their change request with a Round-2 comment. There are four sightings: three defect-notes and one this morning.

- **Brain #1850, 2026-11-19.** Euclid's Round 2 (review 5974216708) is a COMMENT on `c885ace5` with RA-1 `STILL_OPEN`. Submitting it consumed the review request. At 11:34:07Z the Fleet woke the author with `[open-work:route-review]`, "CI is green; request a reviewer for your head", for `c885ace5`.
- **Institution #1018, 2026-11-17, row-4 FM walk.** Euclid's Round 2 COMMENT kept RA-1 `STILL_OPEN`, and the author's cockpit card showed `route-review`. Grace recorded it as the walk's one failed step (3e) on neomjs/neo-agent-institution#490.
- **neo #21192, 2026-11-17.** A Round-2 COMMENT with RA-4 `STILL_OPEN` drew the same wake (Grace's defect-note).

## The Problem

Round 2 is disposition-only and is posted as a COMMENT. An action marked `STILL_OPEN` leaves the Round-1 change request standing, and GitHub's `reviewDecision` stays `CHANGES_REQUESTED`. The author owes that action. Open-work instead tells them to request a review of the head the reviewer just judged: following it spends a review on an unchanged head, and ignoring it teaches seats to ignore `route-review`. It is also the failed step of the row-4 walk, which counts toward FM v1.

## The Architectural Reality

- `ai/services/fleet/openWorkHolder.mjs` `holderOf()` is the one function the open-work projection and the wake path both read.
- `opinions` hold each reviewer's standing approval or change request. The module JSDoc: "Opinions come from each reviewer's standing approval or change request, never their latest review, which a comment can be; a comment on the head is engagement only."
- So a Round-2 COMMENT on the current head adds no opinion. The standing change request sits on an earlier head, so `onHead('CHANGES_REQUESTED')` is false. The request was consumed, so `requested` is empty. The seat-owned route-review branch (`!opinions.some(opinion => opinion.onHead)`) fires.
- `reviewedHere` (seats with a review on the current head) and `stands` (a judgment standing on an earlier head) are already computed. Only the outside-PR path reads `stands`.
- Sibling precedent: closed #1802 corrected another wrong author action in this same function.
- Design authority: #34, The Fix item 5: "A still-open action keeps the original RC authoritative; it does not create a new action list." The module JSDoc also says: "A standing change request or approval outranks an earlier comment".

## The Fix

In `holderOf()`, just before the seat-owned route-review branch, return `{role: 'author', kind: 'address-changes', ids: [owner.seat]}` when all of these hold:
- a seat's change request stands on an earlier head;
- that same seat has a review on the current head;
- no approval is on the head.

Leave the `requested.length` branch ahead of it, so a re-request on an unchanged head (an author's `DEFENDED` handback) still goes to the reviewer. Add one sentence to the module JSDoc.

## Acceptance Criteria

- [ ] AC-1: seat-owned PR, green CI, a seat's change request on an earlier head, the same seat's COMMENT on the current head, nothing requested → author `address-changes`. Red against `dev`'s `openWorkHolder.mjs`, green at head.
- [ ] AC-2: the same, but the COMMENT is on an earlier head because the author pushed past it → `route-review`, as today.
- [ ] AC-3: the same seat re-requested on the current head → reviewer `review`, as today.
- [ ] AC-4: outside PRs are unchanged, and the existing `openWorkHolder` and wake-wiring specs pass.

## Out of Scope

- A reviewer still listed in `reviewRequests` after a COMMENT round on the head (Maryam's defect-note, Institution #962, 2026-11-14). Telling a stale request from a fresh re-request needs request timestamps.
- Parsing review bodies for `STILL_OPEN`.

## Avoided Traps

- **Reading the comment's disposition table.** A standing change request plus that seat's review on the head already says the reviewer kept it.
- **Placing the rule above `requested.length`.** That would take a re-requested review away from the reviewer.

## Related

#759 (the open-work epic, already at 26 subs, so this stands alone) · #34 (Round 2 economics) · #1849 and PR #1850 (today's sighting) · neomjs/neo-agent-institution#490 (row-4 walk, step 3e) · #1802 (sibling holder fix)

Live latest-open + closed sweep at 2026-11-19T12:01Z: latest 20 open read, plus open and closed searches ("route-review comment change request author") and an Institution and Brain search for `route-review` since 11-16. Out of Scope read on hits; no equivalent. #34 governs review economics, not routing. Backlog band: 157 open (red); the owning epic #759 is past 25 subs. A2A in-flight sweep: latest 30 messages, all read-states, at 12:00Z: no claim on this surface. MC sweep ("route-review wake after a Round-2 comment while the change request still stands"): Grace's 11-17 defect-notes, no prior decision. Own-assignment sweep: 6 open, none overlapping. Structure map: `ai/services/fleet/openWorkHolder.mjs`, no placement change.

Decision Record impact: none.

Origin Session ID: 26c9fc6c-df7d-4bb0-8d54-f348f6a9a674

Retrieval Hint: "route-review after a Round-2 comment keeps the change request"

## Timeline

- 2026-11-19T12:04:51Z @neo-opus-ada added the `bug` label
- 2026-11-19T12:04:51Z @neo-opus-ada added the `ai` label
- 2026-11-19T12:04:52Z @neo-opus-ada added the `agent-os` label
- 2026-11-19T12:05:08Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-19T12:09:06Z @neo-opus-ada cross-referenced by PR #1852
### @neo-opus-ada - 2026-11-19T12:20:36Z

**Handover (Ada, session sunset, solo-refresh, 2026-11-19 ~12:25Z).** Owner: still me, so the next session picks this up.

- PR #1852 is open at `9cdb328e` on `ada/1851-round-2-comment-keeps-cr`. CI was pending at sunset.
- Next step: at green, request one non-Claude reviewer. Euclid is on #1850's Round 3, so Sophie or Emmy.
- Evidence so far: the AC-1 spec fails against `dev`'s `openWorkHolder.mjs` and passes at head. The fleet unit dir gives 2523 passed and 3 skipped.
- The change lives inside the existing route-review branch, so only the state this ticket names changes.
- Post-merge: the next Round-2 COMMENT that leaves an action `STILL_OPEN` on a seat's PR should wake its author with `address-changes`. That's the live check, owned by #759.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code · session 26c9fc6c-df7d-4bb0-8d54-f348f6a9a674

- 2026-11-19T13:28:53Z @tobiu referenced in commit `da422ca` - "fix(fleet): a kept change request leaves the author its action (#1851) (#1852)

A Round-2 comment adds no opinion, so a seat-owned PR whose reviewer
kept a change request on an earlier head and commented on this one fell
through to route-review. Inside that branch, a standing change request
whose seat reviewed the current head now yields address-changes."
- 2026-11-19T13:28:55Z @tobiu closed this issue

