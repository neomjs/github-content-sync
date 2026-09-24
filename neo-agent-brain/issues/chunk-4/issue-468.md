---
id: 468
title: 'A claim on a curated issue never wakes a seat: the heartbeat keeps only mention and review_requested notifications'
state: CLOSED
labels:
  - enhancement
  - contributor-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T19:14:55Z'
updatedAt: '2026-09-24T19:16:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/468'
author: neo-opus-grace
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
closedAt: '2026-09-24T19:16:17Z'
---
# A claim on a curated issue never wakes a seat: the heartbeat keeps only mention and review_requested notifications

## Context

A first-time contributor's claim on a curated issue reaches no seat.
- **neomjs/neo#19143:** a real first-timer waited **27 hours** for an assignment. Ada found it by sweeping her curated issues by hand, and assigned them on 2026-09-24.
- **neomjs/neo#19140:** the same day, a claim-then-unclaim followed the same silent path.

The operator's policy (2026-09-24) is why this matters:
- Assignment on external tickets is **behavioral guidance for dedup, not a gate**.
- Unassigned PRs are accepted, and **the first approve-worthy PR for a ticket wins**.
- So a slow assignment penalizes exactly the contributors who ask first.
- A good first issue is normally claimed within about thirty minutes (operator, neomjs/neo#18985 AC-3).

## The Problem

`SwarmHeartbeatService#getGitHubNotifications` fetches `gh api notifications?participating=true` and keeps only `GITHUB_WAKE_NOTIFICATION_REASONS = ['mention', 'review_requested']` (`ai/services/github-workflow/HealthService.mjs:33, :59`).

A comment on an issue the seat authored arrives with reason `author`, or `comment` once the seat has commented, so the filter drops it. Every curated issue so far is authored by a seat, so every claim sat in a notification list that never woke anyone.

## The Fix

Add a second notification source in the heartbeat, modelled on `enrichGitHubNotificationsWithPullRequestState`:

1. **Candidates:** raw `Issue` notifications whose reason is `author` or `comment`.
2. **Resolve:** the issue (state, labels, assignees) and its latest comment (id, `author_association`, `user.type`).
3. **Admit when all of these hold:**
   - the issue is open and unassigned;
   - it is labelled `good first issue` or `hacktoberfest`;
   - the latest comment's `author_association` is `NONE`, `FIRST_TIME_CONTRIBUTOR`, `FIRST_TIMER` or `CONTRIBUTOR`;
   - the commenter is not a `Bot`.
4. **Project** it as a wake notification with the distinct reason `curated-claim`, carrying `{number, commentId, commenter, association}`. It wakes the seat that received the notification.
5. **Dedupe** on `repo#issue@commentId`. Each new claim wakes once, so a second claimant reaches the seat too.

**Out of the payload:** no auto-assignment, and no reply. The seat applies the policy: assign the first human claimant, and answer a second one with "taken" plus an alternative.

**Contract Ledger (T3)**

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `SwarmHeartbeatService` GitHub notification ingestion | neomjs/neo#18985 door design (Ada + Grace, 2026-09-24); operator policy above | admits `curated-claim` wakes per the rule above, alongside the unchanged `mention` / `review_requested` path | a failed issue or comment read omits that candidate and logs a warning, as `resolvePullRequestState` does; the existing reasons are unaffected | JSDoc on the new seam | a unit spec over fixture notifications: a `NONE` claim on an unassigned `good first issue` is admitted; assigned, member-authored, bot-authored, closed and unlabelled cases are rejected; a second comment re-admits once |
| Wake payload (`github-notification.` pulse) | existing projection (`projectWakeNotification`) | adds the `curated-claim` reason and the `issue` echo; existing fields are unchanged | — | JSDoc | the same spec asserts the projected shape |

No AiConfig surface: the label set and the associations are code constants beside `GITHUB_WAKE_NOTIFICATION_REASONS`.

## Acceptance Criteria

- [ ] **AC-1:** A non-member claim comment on an open, unassigned `good first issue` or `hacktoberfest` issue, authored by a seat, produces exactly one `curated-claim` wake for that seat.
- [ ] **AC-2:** Assigned, closed, unlabelled, member-authored and bot-authored cases produce no wake. The `mention` and `review_requested` paths are unchanged.
- [ ] **AC-3:** A later claim comment on the same issue produces one more wake, deduped on `repo#issue@commentId`.
- [ ] **AC-4:** A failed issue or comment resolution omits that candidate and warns, and never drops the rest of the pulse.

## Out of Scope

- Auto-assignment (a `/claim` bot). It would hand reserved first-timer issues to spray agents such as neomjs/neo#19140's.
- Outside contributors' PRs: #427.
- The org-wide state-diff poller: D#19122 (neomjs/neo discussion).
- The contributor-facing wording in `CONTRIBUTING.md` and the curated-issue template: Ada, neomjs/neo#18985.

## Deploy note

The source only helps once the plane runs it: an orchestrator deploy before 2026-10-01, which is the operator's to approve.

## Related

neomjs/neo#18985 · #427 · D#19122 · #106 · neomjs/neo#19140 · neomjs/neo#19143

Sweeps at 19:13Z:
- **Latest open:** the latest 20 open Brain issues; none equivalent.
- **Keywords:** `curated claim`, `good first issue notification`, `notification reason author`, `claim comment wake`, `hacktoberfest` returned only #106, which is a different surface (the durable community-activity substrate).
- **A2A:** this leaf was handed over by @neo-opus-ada at 19:09Z; there is no other claim.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T19:14:56Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T19:14:57Z @neo-opus-grace added the `enhancement` label
- 2026-09-24T19:14:57Z @neo-opus-grace added the `contributor-experience` label
- 2026-09-24T19:14:58Z @neo-opus-grace added the `agent-os` label
### @neo-opus-grace - 2026-09-24T19:16:16Z

Closing as not planned. The premise was delivery through the swarm heartbeat, and the operator ruled that out (2026-09-24, about 19:15Z):
- Heartbeats are disabled on purpose, because they caused too much daytime noise.
- A 20 to 30 minute cadence would still be too slow for claims.

The claim path is being redesigned as event-driven on the neo side: an `issue_comment` responder on curated issues that answers claimants within seconds and wakes no seat. It will be filed under neomjs/neo#18985 once agreed.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

- 2026-09-24T19:16:17Z @neo-opus-grace closed this issue

