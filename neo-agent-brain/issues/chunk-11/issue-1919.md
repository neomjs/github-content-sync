---
id: 1919
title: A parent's last sub closing wakes its steward once with a closeout
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-11-24T20:52:46Z'
updatedAt: '2026-11-24T20:52:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1919'
author: neo-opus-ada
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
# A parent's last sub closing wakes its steward once with a closeout

## Context

The operator ruled on 24 Nov that an issue whose subs are all closed is closed or gets new subs. An audit at 2026-11-24T20:45Z found 33 open parents in that state across neo, Brain, Institution and Skills. Nothing fires when a parent's last sub closes, so the closeout waits until somebody happens to look. The rule itself is going into neomjs/neo-agent-skills `epic-resolution`.

## The Problem

The open-work producer routes PR transitions to the seat that holds the next action (#759). The transition "this parent's last sub just closed" has no holder, so the steward never learns that the next action is theirs.

## The Architectural Reality

- The open-work producer and its holder selection are #759's substrate.
- GitHub reports `subIssuesSummary {total, completed}` per issue.
- Design authority: #759's predicate, "Own-work transitions reach the seat that holds the next action".

## The Fix

When a watched repository's open issue has at least one sub-issue and every sub is closed, the producer emits one `closeout` wake. The holder is the issue's assignee, or its author when unassigned. The wake names the rule: close it, or file the missing subs.

The wake is deduplicated on the transition. A sub that reopens, or a new open sub, clears it.

## Acceptance Criteria

- [ ] Closing the last open sub produces exactly one `closeout` wake to the parent's holder.
- [ ] No wake while any sub is open. A reopened sub or a new open sub clears the pending closeout.
- [ ] Unit tests cover the selector and the deduplication.

## Out of Scope

The skill rule (neomjs/neo-agent-skills `epic-resolution`) and any automatic closing.

## Related

#759 (parent).

Live latest-open + closed sweep: checked the latest 20 open issues plus an open and closed search (`closeout last sub closed parent epic wake`); #759 owns own-work transitions and no closeout kind exists. 155 open (red band), so this is folded under #759 as a sub.
A2A in-flight claim sweep: no claim.
MC sweep: no prior decision found.
Own-assignment sweep: none on the open-work producer.

unowned-rationale: the open-work producer's maintainers (#759's family) pick it up; Ada reviews.

Origin Session ID: 36593ade-49d5-4bf2-92e6-6e86d12beb9b
Retrieval Hint: "open-work closeout wake parent last sub closed steward"

## Timeline

- 2026-11-24T20:52:47Z @neo-opus-ada added the `enhancement` label
- 2026-11-24T20:52:47Z @neo-opus-ada added the `ai` label
- 2026-11-24T20:52:51Z @neo-opus-ada added parent issue #759

