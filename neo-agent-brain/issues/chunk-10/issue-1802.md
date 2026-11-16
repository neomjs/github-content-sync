---
id: 1802
title: Fleet tells the author to repair CI when the failing check reports a merge conflict
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-16T13:31:39Z'
updatedAt: '2026-11-16T14:19:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1802'
author: neo-gpt-emmy
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
closedAt: '2026-11-16T14:19:57Z'
---
# Fleet tells the author to repair CI when the failing check reports a merge conflict

## Context

Engine [#21101](https://github.com/neomjs/neo/pull/21101) at `6f7f151199fa524b7d9861688ca2495f70b7369a` had an independent approval on that head, native `CONFLICTING`, and one failing status, `review-admission/mergeability`. The rest of its checks had passed or skipped. Installed Fleet and the current Brain selector classified the next action as `repair-ci`. Rerunning a job cannot integrate the branch.

## The Problem

The synthetic mergeability check participates in the CI rollup. The red branch in `ai/services/fleet/openWorkHolder.mjs` wins before its approved conflict branch. The same complete input with only the CI summary changed to green correctly selects `rebase`.

## The Architectural Reality

Design authority: the existing holder selector sends an independently approved conflicted head for integration. The existing [ownership change](https://github.com/neomjs/neo-agent-brain/pull/1791) also routes appropriate outside integration to the maintainer rotation. The cockpit and wakes consume this same decision. Parent outcome: #759.

## The Fix

Use the native conflict and exact-head approval to choose the actionable integration step even when the integration check itself makes CI red. Retain real test failures in the row and account for them after integration. Preserve current ownership, explicit change requests, pending approvals of fork runs, drafts, partial evidence and ineligible third-party reviews. Avoid reinterpreting GitHub's statuses globally.

## Acceptance Criteria

- [ ] Replay the observed approved maintainer PR with native conflict and only the mergeability status red. Cockpit and wake selection name the author and the integration action.
- [ ] An eligible approved outside PR with the same evidence goes to the existing integration rotation. A real red on a mergeable PR continues to require CI repair. Exercise a mixed red/conflict and a current change request to make precedence explicit.
- [ ] Draft, incomplete and stale-review controls retain their existing action semantics. Demonstrate the fix by replaying a representative row through the normal holder projection. Live post-update verification belongs to #759.

Origin Session ID: 289da399-da26-4b13-ad71-dae9311704bf

## Timeline

- 2026-11-16T13:31:40Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-16T13:31:41Z @neo-gpt-emmy added the `bug` label
- 2026-11-16T13:31:41Z @neo-gpt-emmy added the `ai` label
- 2026-11-16T13:31:41Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-16T13:38:41Z @neo-gpt-emmy cross-referenced by PR #1804
- 2026-11-16T14:00:43Z @neo-gpt-emmy referenced in commit `a101aa8` - "fix(fleet): localize independent integration approval and include unowned heads (#1802)"
- 2026-11-16T14:19:57Z @tobiu referenced in commit `5a9c679` - "fix(fleet): route approved conflicts to integration when CI is red (#1802) (#1804)

* fix(fleet): route approved conflicts to integration when CI is red (#1802)

* fix(fleet): localize independent integration approval and include unowned heads (#1802)"
- 2026-11-16T14:19:58Z @tobiu closed this issue

