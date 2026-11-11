---
id: 350
title: An outside contributor's pending workflow runs have no owner in the rules
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T19:32:09Z'
updatedAt: '2026-11-11T20:14:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/350'
author: neo-opus-grace
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
closedAt: '2026-11-11T20:14:51Z'
---
# An outside contributor's pending workflow runs have no owner in the rules

## Context

The operator, 2026-11-11, after approving the pending runs on #338 himself: "you as a maintainer like all other peers can do this without me". A seat had sent the approval to him twice that day, as a task from the Fleet's `approve-runs` wake. Its reason was a memory, written after one classifier hold, that classed fork CI approval with merging.

## The Problem

No skill says who approves the workflow runs that GitHub holds for an outside contributor's pull request (`action_required`), or what to check first. Each seat has to infer it. One inference sends a maintainer act to the operator. The other approves code it never read.

## The Architectural Reality

- `.agents/skills/pr-review/references/pr-review-guide.md:20` is the rule for external and first-time contributors: "same rigor, warmer register". It has nothing on their runs.
- Design authority: `AGENTS.md` §swarm_topology_anchor, Tier 4: "Ask the human only for human-owned domains (merge, credentials, subjective aesthetics) or operator-surfaced intent clarification." Run approval is not on that list, so it is a maintainer act.
- The Fleet routes `approve-runs` to the maintainer rotation (neomjs/neo-agent-brain `ai/services/fleet/openWorkHolder.mjs:41`). So the producer already treats the act as a maintainer's.

## The Fix

`pr-review-guide.md:20` gains one sentence. A maintainer approves an outside contributor's pending runs once the diff against the base changes nothing the runs execute; otherwise they read those changes first. Then the package version is bumped.

## Acceptance Criteria

- [ ] The external-contributor rule names the act, who owns it, and the check: `.github/**`, package scripts, lockfile entries beyond the version, source, and build or test code. Source is on the list because consumer `pull_request` runs execute it (neomjs/neo `test.yml` runs the unit suites over `src/**`).
- [ ] The package version is bumped, so consumers receive the rule.

## Out of Scope

The Fleet's `approve-runs` wake text. Whether that message names the act and its check is neomjs/neo-agent-brain#1607, which moves its composition into the plane.

## Related

#338 · neomjs/neo-agent-brain#1607 · neomjs/neo-agent-brain#759

Decision Record impact: none.

Live latest-open + closed sweep: the latest 12 of 34 open (orange band), plus an all-state search (`workflow runs approve outside contributor fork`; no hits), at 2026-11-11T19:31Z. A2A in-flight claim sweep: the latest messages at 19:22Z show no claim on this surface. MC sweep: only my own trail, which records the wrong classification. Own-assignment sweep: #289 and #258, no overlap.

handoff: @neo-opus-grace self-assigns.

Origin Session ID: 2d4f1520-5217-4756-9013-970e443dcb03

Retrieval Hint: "who approves an outside contributor's pending workflow runs; maintainer act, check the diff for executed code first"


## Timeline

- 2026-11-11T19:32:10Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T19:32:11Z @neo-opus-grace added the `ai` label
- 2026-11-11T19:32:14Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T19:32:19Z @github-actions added the `needs-design-authority` label
- 2026-11-11T19:35:15Z @neo-opus-grace cross-referenced by PR #352
- 2026-11-11T20:03:35Z @neo-opus-grace referenced in commit `d4eb30f` - "fix(pr-review): the run-approval check names source code, which consumer CI executes too (#350)"
- 2026-11-11T20:04:38Z @github-actions removed the `needs-design-authority` label
- 2026-11-11T20:12:21Z @neo-opus-grace cross-referenced by #353
- 2026-11-11T20:14:51Z @tobiu referenced in commit `12ef5d8` - "feat(pr-review): any maintainer approves an outside contributor's pending runs after a diff check (#350) (#352)

* feat(pr-review): any maintainer approves an outside contributor's pending runs after a diff check (#350)

No skill said who approves the workflow runs GitHub holds for an outside contributor's pull request, so a seat sent the act to the operator. The external-contributor rule now names the owner and the check: approve once the diff touches nothing the runs execute, otherwise read those changes first. Version 1.4.2.

* fix(pr-review): the run-approval check names source code, which consumer CI executes too (#350)"
- 2026-11-11T20:14:53Z @tobiu closed this issue
- 2026-11-11T20:21:21Z @shounakjoshi88-a11y cross-referenced by PR #338
- 2026-11-11T21:15:30Z @neo-opus-grace cross-referenced by #20794

