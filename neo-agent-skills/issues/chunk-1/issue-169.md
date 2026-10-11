---
id: 169
title: 'Withdrawing an approval should be a Request Changes review, with the hold token as fallback'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-11T00:04:32Z'
updatedAt: '2026-10-11T01:21:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/169'
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
closedAt: '2026-10-11T01:19:19Z'
---
# Withdrawing an approval should be a Request Changes review, with the hold token as fallback

## Context

Design authority: the operator, 2026-10-10, after neo #19574's hold. Relayed in neomjs/neo-agent-brain#981: a merge hold "might be insufficient; changing the GitHub review state to Changes Requested might win." Vega forked the doc half to me as its author (Tier 2.5, A2A 2026-10-11 00:03Z); the validator half is neomjs/neo-agent-brain#981.

## The Problem

`pr-review/references/merge-hold-tokens.md` and `pr-review-guide.md §9.2` tell a reviewer to withdraw an approval with a `[MERGE_HOLD]` / `[RE_REVIEW_HOLD]` comment. That comment changes no GitHub state, counts as no round, and leaves the clearing review's form undecided: Round 2 has no Round 1 to cite, so the full template is repeated. The doc also says dismissing the approval through GitHub's UI "also works", but a reviewer cannot dismiss their own review. A newer review is GitHub's own way to replace it.

## The Architectural Reality

- `pr-review-guide.md` §9.2 points to `merge-hold-tokens.md`, which is load-on-demand.
- `validateMergeReady` reads the tokens (the Engine-era #17608 / #17671); a `CHANGES_REQUESTED` review is GitHub's own `reviewDecision` input and the budget's counted round (§6.3, one ordinary RC per family).
- neomjs/neo-agent-brain#981 makes the Round-2 relation gate hold on `update` as on `create`.

## The Fix

1. `merge-hold-tokens.md` leads with the review: withdraw by submitting `CHANGES_REQUESTED`, whose Required Actions a Round 2 later dispositions. The token stays for when no round can open (the family's round is spent), and a hold cleared without a round takes the full template. The dismissal sentence goes.
2. §9.2 names the review first and the token as the fallback, in no more bytes than today.

## Acceptance Criteria

- [x] AC-1 The doc names the review as the instrument, the token's remaining case, and the clearing form after a hold-only withdrawal; it no longer claims a reviewer can dismiss their own review.
- [x] AC-2 §9.2 matches, and the corpus lint (including the guide's combined budget) passes.

## Out of Scope

The validator (neomjs/neo-agent-brain#981); counting a hold comment as a round (rejected there).

## Related

neomjs/neo-agent-brain#981, neomjs/neo#19574, #76.

Sweeps: live latest-open sweep of this repository's 20 newest open issues at 00:04Z, no equivalent (#76 is a stale approval after a push, a different trigger) · A2A: Vega's fork is the only claim · MC: the operator's words via #981 · Own-assignment: none open.

Origin Session ID: afd79583-ff45-4f2c-96a5-08549257af36
Retrieval Hint: "withdraw approval CHANGES_REQUESTED review merge hold token fallback clearing form"

## Timeline

- 2026-10-11T00:04:33Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-11T00:04:34Z @neo-opus-grace added the `enhancement` label
- 2026-10-11T00:04:34Z @neo-opus-grace added the `ai` label
- 2026-10-11T00:10:03Z @neo-opus-grace cross-referenced by PR #170
- 2026-10-11T00:25:05Z @neo-opus-grace referenced in commit `2852d00` - "docs(pr-review): the hold token also covers a moved head with no defect established yet (#169)"
- 2026-10-11T01:19:19Z @tobiu referenced in commit `7d7c2de` - "docs(pr-review): withdraw an approval with a Request Changes review; the hold token is the fallback (#169) (#170)

* docs(pr-review): withdraw an approval with a Request Changes review; the hold token is the fallback (#169)

* docs(pr-review): the hold token also covers a moved head with no defect established yet (#169)"
- 2026-10-11T01:19:19Z @tobiu closed this issue

