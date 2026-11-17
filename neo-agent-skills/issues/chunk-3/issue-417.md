---
id: 417
title: A mechanical rebase keeps its approval; an outside author's push does not
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
  - needs-design-authority
assignees:
  - neo-opus-ada
createdAt: '2026-11-17T11:23:48Z'
updatedAt: '2026-11-17T11:31:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/417'
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
---
# A mechanical rebase keeps its approval; an outside author's push does not

## Context

The operator, 2026-11-17, after an author re-requested a reviewer (with 13% of his weekly budget left) on three approved PRs she had rebased: *"was there a single occurrence so far, where a reviewer said RC after a rebase?"* The census answer was no. Over the last 100 merged `neomjs/neo` PRs (Nov 15 03:50Z – Nov 17 11:00Z), no reviewer requested changes after a rebase-only change. The one approve→RC sequence (neo#21093) followed a content rewrite. 10 PRs carried approvals on more than one head, 16 extra approvals in all, an upper bound for what rebases cost. The operator had merged neo#21220 on its dev-merge head with the approval still on the pre-merge commit.

His ruling: the rule applies to every peer. His exception: outside contributors, *"in theory malicious ones could otherwise sneak in code we do not want on a 'rebase' change."*

## The Problem

No skill says that a base update is not a re-review event. After an author rebases an approved PR, both signals they see push toward a re-request: the Fleet's `[open-work:route-review]` wake after the head moves (neo#21227 at 10:52Z), and the merge-readiness advisory "approval anchor is stale … confirm the delta is content-free … or re-request review". The re-request is then what blocks strict merge-readiness ("outstanding requested reviewer(s)").

## The Architectural Reality

- `pull-request/references/ci-green-review-routing.md` §4 "Re-Review Requests" covers re-requests after addressing feedback; base updates are not mentioned.
- `pull-request/references/pull-request-workflow.md` §10 "Maintenance" covers a maintainer merging the base into a contributor's head; it says nothing about review.
- Stacked PRs already need their approval on the dev-rebased candidate (§6.1). This ticket leaves that unchanged.
- #76 (open) asks for a gate that notices an approval the head no longer matches. Its AC-2 ("fails when the head is advanced by one commit") would mandate the re-approvals this ruling drops.

Design authority: the operator's ruling above (2026-11-17).

## The Fix

In `ci-green-review-routing.md` §4, loaded at the re-review trigger, add the rule. A base update (rebase or dev merge) of an approved PR is not a re-review event when `git range-diff` between the approved and the new commits shows only context lines, generated files, or lines forced by the base's own changes. The author posts that delta on the PR and requests no one, and re-requests only when reviewed content changed. Exception: on an outside contributor's PR, any push by its author after approval re-enters review of the delta, because an untrusted "rebase" can carry content. A maintainer's own base merge on such a PR (§10) stays a mechanical delta. Keep the addition small and compress the surrounding text for a net loaded-bytes reduction.

## Acceptance Criteria

- [ ] §4 states the base-update rule, the `range-diff` delta note and when to re-request.
- [ ] The outside-author exception is stated where an author or reviewer of an outside PR reads it.
- [ ] #76 records the ruling's disposition, so its gate keeps a mechanically rebased maintainer approval.
- [ ] The PR nets loaded bytes down or cites a decay rationale (§self_evolving_systems).

## Out of Scope

The approval-vs-head gate itself (#76); the Fleet's route-review producer (defect note 71312615); `validateMergeReady`'s advisory wording (Brain).

## Related

#76, #169, #399; neo#21227 (the specimen and its delta note: https://github.com/neomjs/neo/pull/21227#issuecomment-6946432535); neo#21093 (the content-rewrite RC).

Live latest-open + closed sweep at 11:20Z: the latest 20 open read, plus open and closed "rebase review / re-review / stale approval / re-approval" searches. #76 is related (the gate) and nothing duplicates this. Backlog 39 open (orange): no ticket owns the author rule; #76's scope is the gate.

Origin Session ID: ea74f891-dcaa-40d2-b7b8-916352c11609

## Timeline

- 2026-11-17T11:23:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-17T11:23:50Z @neo-opus-ada added the `enhancement` label
- 2026-11-17T11:23:50Z @neo-opus-ada added the `ai` label
- 2026-11-17T11:23:50Z @neo-opus-ada added the `agent-os` label
- 2026-11-17T11:24:01Z @neo-opus-ada cross-referenced by #76
- 2026-11-17T11:25:04Z @github-actions added the `needs-design-authority` label
- 2026-11-17T11:26:05Z @neo-opus-ada cross-referenced by PR #21228
### @neo-opus-ada - 2026-11-17T11:31:01Z

**Handover, session sunset (owner: @neo-opus-ada, picked up by her next session).** Not started; nothing is branched.

- Where: `.agents/skills/pull-request/references/ci-green-review-routing.md` §4 "Re-Review Requests" (lines ~81–89 at `3c1096d`). Add the base-update rule after its first paragraph. Keep it short and compress §4's existing text for the net-bytes AC.
- What the rule says: after a rebase or dev merge of an approved PR, run `git range-diff <old-base>..<old-head> <new-base>..<new-head>`. If only context lines, generated files or lines forced by the base's own changes differ, post that delta on the PR and request no one. Otherwise re-request. On an outside contributor's PR, any push by its author after approval re-enters review; a maintainer's own base merge there stays mechanical.
- Worked examples: neo#21227 (https://github.com/neomjs/neo/pull/21227#issuecomment-6946432535) and neo#21228 (https://github.com/neomjs/neo/pull/21228#issuecomment-6946702268). Both merged or are mergeable without a re-review.
- Before editing: read `create-skill` and `turn-memory-pre-flight` (substrate PR), then the `pull-request` workflow for the PR itself.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-17T12:03:47Z @neo-opus-ada cross-referenced by PR #418
- 2026-11-17T12:10:14Z @neo-opus-ada referenced in commit `7c69c70` - "chore(release): neo-agent-skills 1.10.3 (#417)"

