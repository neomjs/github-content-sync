---
id: 1619
title: A merge wakes the seats whose blocked tickets it unblocks
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T23:00:49Z'
updatedAt: '2026-11-11T23:00:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1619'
author: neo-opus-grace
commentsCount: 0
parentIssue: 759
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
# A merge wakes the seats whose blocked tickets it unblocks

## Context

On 2026-11-11 the operator confirmed that merge messages should wake: "in case one merge unblocks other lanes and peers are idle waiting for this to happen." Since #1610, every open-work message wakes its recipient, the `post-merge` kind included.

A `post-merge` wake goes to one seat, the merged PR's author (`ai/services/fleet/openWorkWakes.mjs:213`: `wakeOf(transition, transition.owner.seat, 'author', 'post-merge', …)`). The seats the operator means hold tickets that the merge unblocked, and nothing wakes them. Memory Core shows the cost recurring: seats park "until PR #11704 merges to unblock the next phase" and stay parked after it merges.

## The Problem

A merge that unblocks another lane is invisible to that lane's owner until a human or a heartbeat happens to look. The open-work producer reads no dependency: nothing under `ai/services/fleet/` names `blocked_by`, `blockedBy` or `blocking`.

## The Architectural Reality

- The producer already observes each merge as a terminal transition (`openWorkReducer.mjs`, `transition(…, 'merged', 'open', 'merged', pulse)`), and the round turns it into the author's `post-merge` wake.
- GitHub records the dependency natively, and one query resolves it. Receipt at 2026-11-11T23:0xZ: neomjs/neo PR #20791 (`merged=true`) → `closingIssuesReferences` #20788 → `blocking` #20790 (OPEN, assignee `neo-opus-grace`).
- #1607 (PR #1613) moves open-work wakes to a plane intake that composes fixed text from tokens and sends as `@system`. A new kind rides it.

## The Fix

On a merged transition, the producer also resolves what the merge unblocks:
1. Merged PR → `closingIssuesReferences` → each closed issue's `blocking` issues that are still open.
2. A blocked issue counts as unblocked only when its `blockedBy` holds no other open issue. One merge of two blockers wakes nobody yet.
3. Each assignee of an unblocked issue that maps to a registered seat gets one `unblocked` wake, with fixed text naming the issue and the merged PR, through the #1607 intake. The episode is keyed by issue and merge, so a re-observed merge does not wake twice.
4. Cost: one query per merged PR, outside the per-pulse snapshot.

## Acceptance Criteria

- [ ] Unit, on the producer's resolution and plan:
  - A merge that closes the last open blocker of an assigned issue plans one `unblocked` wake to the assignee seat.
  - A merge that leaves another open blocker plans none.
  - An unassigned blocked issue, or an assignee with no seat, plans none.
  - Re-observing the same merge plans nothing new.
- [ ] Unit, on the intake contract: `unblocked` validates and composes like the other kinds (fixed text, tokens only, normal priority, wake on).
- [ ] *(post-merge, installed)* A real merge that closes a blocker wakes the blocked ticket's assignee seat, as seen in the FM feed.

## Out of Scope

- Stacked pull requests whose base was the merged branch: their authors' retarget is a different action.
- Unassigned blocked tickets, and dependencies recorded only in prose (no `blocked_by` relationship).
- The author's own `post-merge` wake, which stays as it is.

## Related

Parent #759 · #1610 (every kind wakes) · #1607 / PR #1613 (the intake) · #807 (escalations, same producer)

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 open (red band; the owning epic is #759, so this is filed as its leaf) and a state-all search (`unblock wake OR "blocked_by" merged wake OR dependents merge wake`; no equivalent) at 2026-11-11T23:00Z. A2A in-flight claim sweep: the latest 12 messages, back to 22:09Z, carry no claim on this surface. MC sweep: the waiting pattern above, no prior decision. Own-assignment sweep: #807, #1607, #1614, no overlap.

handoff: @neo-opus-grace self-assigns, as owner of #759.

Origin Session ID: 15ea444b-707c-4458-bba5-79b4f61fb280

Retrieval Hint: "merge wakes the assignees of tickets it unblocks; blocked_by closingIssuesReferences blocking; idle peers waiting for a merge"

## Timeline

- 2026-11-11T23:00:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T23:00:51Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T23:00:51Z @neo-opus-grace added the `ai` label
- 2026-11-11T23:00:51Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T23:01:05Z @neo-opus-grace added parent issue #759
- 2026-11-12T00:15:49Z @neo-opus-grace cross-referenced by #1620

