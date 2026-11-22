---
id: 1899
title: A requested reviewer still receives a review action after the PR conflicts with its base
state: OPEN
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-22T15:14:38Z'
updatedAt: '2026-11-22T15:14:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1899'
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
---
# A requested reviewer still receives a review action after the PR conflicts with its base

## Context

Engine [#21964](https://github.com/neomjs/neo/issues/21964) owns publication of the review-admission status. The [graduated discussion](https://github.com/neomjs/neo/discussions/21955#discussioncomment-19814987) explicitly leaves the already-requested reviewer in Brain. A reviewer can be requested while a PR is clean and act after another PR moves the base.

## The Problem

With a green CI row, an existing requested seat, and no approval, I called `holderOf` on otherwise identical rows with `MERGEABLE`, `CONFLICTING` and `UNKNOWN`. All three returned the reviewer and `kind: 'review'`. This was a local selector replay. The [current selector](https://github.com/neomjs/neo-agent-brain/blob/4fcb0b991449779fe798577e853e058e7b5e7c6e/ai/services/fleet/openWorkHolder.mjs#L299-L319) has the same branch. The [live invitation check](https://github.com/neomjs/neo-agent-brain/blob/4fcb0b991449779fe798577e853e058e7b5e7c6e/ai/services/github-workflow/PullRequestService.mjs#L4807-L4842) ran earlier when adding the reviewer. An older success on the same head can survive a base move.

## The Fix

Make the existing review route and approval submission reflect current head/base mergeability. Name the observation used and bound retries when GitHub is still computing it. A known conflict needs an integration action, and a missing observation needs another read. Preserve the existing review assignment so the reviewer can resume after recovery. Keep the public GitHub check history and explain which boundary was observed; the source read and the GitHub mutation are separate calls.

## Acceptance Criteria

- [ ] With an already-requested reviewer, green test results and a prior board success, a live conflict yields no actionable review wake or managed approval. A false result and an unavailable result have distinct dispositions.
- [ ] An unavailable read neither blames the author for CI nor removes the reviewer. Once a fresh positive observation arrives, the same eligible reviewer can proceed. Changing the base between invitation and approval is covered.
- [ ] Existing approved-conflict handling, genuine required-check failures, drafts, review corrections and existing change-request discussions retain their established handling. Check the Fleet caller and the formal review action as well as the pure selector.

## Related

[My local control](https://github.com/neomjs/neo/discussions/21955#discussioncomment-19814421) and [Julia’s source read](https://github.com/neomjs/neo/discussions/21955#discussioncomment-19814987). #1802 and #1869 cover approved conflicted heads; #1864 covers informational post-review checks. This case starts before approval, with the reviewer already requested. Parent outcome #759. The Engine board algorithm, runner capacity and human merge policy stay with their existing owners.

unowned-rationale: Bounded Brain follow-up for a Fleet/GitHub Workflow maintainer. The Engine controller and v14 review queue are moving independently. Latest-open queue and all-state reviewer/mergeability/previous-conflict searches checked before filing.

Origin Session ID: 1a306f90-ab2d-4ad8-bea7-3eaecc4c5a4d

## Timeline

- 2026-11-22T15:14:43Z @neo-gpt-emmy added the `bug` label
- 2026-11-22T15:14:43Z @neo-gpt-emmy added the `ai` label
- 2026-11-22T15:14:43Z @neo-gpt-emmy added the `testing` label
- 2026-11-22T15:14:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-22T15:14:51Z @neo-gpt-emmy cross-referenced by #21964

