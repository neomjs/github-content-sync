---
id: 1492
title: validate_pr_review_body cannot predict the Round-2 carry refusal
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-05T19:41:12Z'
updatedAt: '2026-11-05T19:41:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1492'
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
# validate_pr_review_body cannot predict the Round-2 carry refusal

## Context

On 2026-11-05 my Round-2 review of neomjs/neo#20352 passed `validate_pr_review_body` (`valid: true`), and `manage_pr_review` then refused it. The rows did not carry the Round-1 Required Actions verbatim: the gate wants each checkbox line including its bold `**RA-n:**` prefix, and only the first line of a multi-line action. I learned the rule from the refusal and had to resubmit.

The validator's description says it "does not predict the submit gate", which is accurate. But the Round-2 carry rule is the one check a reviewer can't verify by reading the template, because it depends on the prior round's text.

## The Architectural Reality

- `PullRequestService#validatePrReviewBody({body})` (`ai/services/github-workflow/PullRequestService.mjs`) runs only `getPrReviewTemplateValidationFailure`, which checks body shape.
- The carry check is `getRound2DispositionRelationFailure({body, reviews, state})`, in the same file. It needs the PR's prior reviews, which `managePrReview` fetches and the validator never does.

## The Fix

Give `validate_pr_review_body` optional `pr_number` and `repo` arguments. When they are given and the body is a Round-2 disposition (`isRound2PrReview`), fetch the PR's reviews the same way `managePrReview` does and run `getRound2DispositionRelationFailure`. The tool stays read-only. Without `pr_number`, nothing changes.

## Acceptance Criteria

- [ ] With `pr_number`, a Round-2 body whose rows don't carry the prior actions verbatim returns `valid: false`, with the same per-row diagnostic `manage_pr_review` gives.
- [ ] With `pr_number`, a correctly carried Round-2 body returns `valid: true`.
- [ ] Without `pr_number`, behaviour is unchanged (shape only).
- [ ] The tool never posts or edits a review.

## Out of Scope

- Changing the carry rule itself.
- Predicting the review-budget gate.

Live latest-open + closed sweep: checked the latest 10 open Brain issues, plus "validate_pr_review_body round 2" and "validatePrReviewBody verbatim". No equivalent (2026-11-05 19:45Z). A2A in-flight sweep: none. Own-assignment sweep: none.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-05T19:41:14Z @neo-opus-ada added the `enhancement` label
- 2026-11-05T19:41:14Z @neo-opus-ada added the `ai` label
- 2026-11-05T19:41:14Z @neo-opus-ada added the `agent-os` label
### @neo-opus-ada - 2026-11-05T19:41:26Z

unowned-rationale: ready for any seat. It is a read-only extension of `validatePrReviewBody` with one fetch, mirroring `managePrReview`'s reviews read. I'm taking #1491 first.

⚖️ Ada


