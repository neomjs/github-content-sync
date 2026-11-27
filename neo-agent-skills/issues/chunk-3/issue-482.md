---
id: 482
title: Shared PR guards compare a merge checkout with a stale base
state: OPEN
labels:
  - bug
  - ai
  - needs-design-authority
assignees: []
createdAt: '2026-11-27T04:29:18Z'
updatedAt: '2026-11-27T04:29:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/482'
author: neo-gpt-sophie
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
---
# Shared PR guards compare a merge checkout with a stale base

## Context

A body edit on [Engine #22111](https://github.com/neomjs/neo/pull/22111) reran `baseline / guards` on an unchanged docs-only head. The [hosted job](https://github.com/neomjs/neo/actions/runs/45939026740/job/140308794757) rejected new size-baseline enrollment for `src/main/DomAccess.mjs`, which the article did not change.

## The Problem

The job checked out `30a21c7af183f395c368928cff88bcf1eec5fb3b`. Its first parent is `11ee760ce34fe4233ec71475a28cd8f26fae835b`, and its second parent is article head `812acbc7853ba62518742fe86f418c14f346bdf0`. It passed event base `c092605497ef66a9a399ba1c1c70824e548ab27e` to the guard.

On that exact checked-out commit, `node buildScripts/util/check-file-sizes.mjs --base c092605497` reproduced one violation. Using `--base 11ee760ce3` reported zero. The article changes two docs files relative to the merge parent. The DomAccess growth came from [Engine #22246](https://github.com/neomjs/neo/pull/22246), which carried its own declaration.

## The Architectural Reality

[The reusable `guards` job](https://github.com/neomjs/neo-agent-skills/blob/v2.9.4/.github/workflows/reusable-pr-baseline.yml#L441-L489) uses the default pull-request merge checkout and injects the event base SHA as `BASE_SHA`. The separate `PR baseline` job checks out the contributor head.

Design authority: [Skills #436](https://github.com/neomjs/neo-agent-skills/issues/436) gives the caller guards the PR base in history and the live body. [Skills #427](https://github.com/neomjs/neo-agent-skills/issues/427) requires a real same-head evaluation on body edits.

## The Fix

In `guards`, resolve `BASE_SHA` from the checked-out synthetic merge. Confirm the second parent is the event head and the first parent is available. Pass that verified first parent to the caller guards. Log the checked-out SHA and both parents. Reject a checkout with unverified coordinates before running the caller commands.

## Contract Ledger Matrix

| Surface | Authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| `guards` `BASE_SHA` | Checkout and input contract from #436 | For a verified synthetic PR merge, use the first parent of the actual checkout | Fail with the observed coordinates when the checkout cannot be paired | Reusable workflow regression with an old event base and a newer merge parent |
| Live `BODY_FILE` and exact-head `PR baseline` job | #427 and #436 | Continue reading the live body on each attempt and preserve the separate exact-head checks | Existing job behavior | Workflow-contract checks |

## Acceptance Criteria

- [ ] A test fixture pairs a stale event base with a synthetic merge against a newer base and shows the consumer guard receives the actual first parent. The fixture detects the old behavior.
- [ ] Validate the checked-out head pair before guards run, including a mismatch case.
- [ ] Keep body-edit triggering, live body reads, and read-only permissions covered by the existing workflow-contract tests.

## Post-Merge Validation

- [ ] Verify a consumer run using a published tag, with an unchanged PR head and differing event base, accepts inherited growth and still rejects growth introduced by the PR.

## Out of Scope

The article’s editorial review and the Engine file-size guard’s thresholds.

Related: #436, #427, neomjs/neo#21504, neomjs/neo#22111.

Decision Record impact: none.

Live latest-open + closed sweep: checked the latest 20 open Skills issues and open and closed searches for baseline guards and synthetic merge base; 47 open. Prior #436 and #427 are linked above. A2A: no matching claim in the latest 30 messages. Memory and own-assignment searches surfaced no open equivalent.

Handoff: @neo-opus-grace, maintainer of the existing guards contract. Shared CI workflow repair.

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822
Retrieval Hint: "PR Baseline edited synthetic merge first parent BASE_SHA DomAccess false enrollment"


## Timeline

- 2026-11-27T04:29:20Z @neo-gpt-sophie added the `bug` label
- 2026-11-27T04:29:20Z @neo-gpt-sophie added the `ai` label
- 2026-11-27T04:29:28Z @github-actions added the `needs-design-authority` label

