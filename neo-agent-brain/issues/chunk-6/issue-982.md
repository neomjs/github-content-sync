---
id: 982
title: Equal timestamps silently clear reviewer merge holds
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-10-11T00:31:16Z'
updatedAt: '2026-10-11T01:18:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/982'
author: neo-gpt
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
closedAt: '2026-10-11T01:18:15Z'
---
# Equal timestamps silently clear reviewer merge holds

## Context

While reviewing Skills `#170`, an exact-source control exposed a separate existing failure in Brain's merge-hold reader. At `98e52e9e067fa555632c9547d29527ba17b95a2d`, a prior approving reviewer posts a hold, then a review is represented with the **same timestamp** as that hold. `resolveMergeHold()` returns `held: false` for both `COMMENTED` and `APPROVED` states. The representation establishes no later ordering.

Design authority: the owning [JSDoc](https://github.com/neomjs/neo-agent-brain/blob/98e52e9e067fa555632c9547d29527ba17b95a2d/ai/services/github-workflow/shared/mergeHoldTokens.mjs#L185-L189) says “a review submitted at the same instant as the hold cannot be assumed to supersede it” and requires strictly newer clearance.

## The Problem

The current indices and comparisons silently weaken that contract. Standing uses the latest approval, so an equal-timestamp approval masks an earlier approval that already granted standing. Clearance retains a hold only while `comment.createdAt > latestReview`, which treats equality as clearance. This is a reproduced pure-reader defect, not a claim that a particular live PR was merged past a hold.

Executed controls against the complete immutable module (Git blob `fed88d4a476997512c2260157e9df741e24490fb`):

| History after a strictly prior approval | Actual `held` | Required |
| --- | --- | --- |
| Hold only | true | true |
| Holder `COMMENTED` at hold timestamp | false | true |
| Holder `APPROVED` at hold timestamp | false | true |
| Holder review strictly later | false | false |
| Other reviewer strictly later | true | true |

## The Architectural Reality

`ai/services/github-workflow/shared/mergeHoldTokens.mjs` owns this pure timeline reduction; `PullRequestService` and the lifecycle merge-readiness path consume it. The existing `test/playwright/unit/ai/services/github-workflow/shared/mergeHoldTokens.spec.mjs` is the sibling test home. No new module, config, durable state or protocol is needed.

The helper's standing rule still requires a strictly prior approval from the holder. Its clearance rule requires a strictly later submitted review from that same holder. A first approval tied with the hold does not alone prove standing. Original-comment edits remain the documented mutable-source limitation.

## The Fix

Repair the existing reader's two timeline questions: preserve evidence of a strictly earlier approval even when a later approval ties the hold; clear an established hold only on a strictly newer holder review. Add discriminating equal-time controls to the existing spec, keeping the older/newer and other-reviewer controls.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `resolveMergeHold({comments, reviews, truncated})` | owning standing/clearance JSDoc | an established hold survives equal timestamps; strictly later same-holder review clears | no strictly prior approval gives no standing; truncated-negative remains unresolved | clarify index/comparison comments without changing the contract | same-time COMMENTED/APPROVED red-first controls plus current positive/negative arms |

Decision Record impact: none — enforce the documented contract, with no new authority or event schema.

## Acceptance Criteria

- [ ] An established hold survives a same-holder `COMMENTED` review at exactly the hold timestamp.
- [ ] An established hold survives a same-holder `APPROVED` review at exactly that timestamp, without losing its earlier approving witness.
- [ ] Strictly newer holder reviews clear; strictly older reviews and other reviewers do not; a tied first-only approval does not confer standing.
- [ ] Existing token classification, truncation and caller tests remain green with no API change.

## Out of Scope

Review-family budgets; Round-2 create/update parity (Vega's `#981`); token vocabulary; immutable issuance history; timestamp-format redesign; GitHub review dismissal permissions.

## Avoided Traps

A tied timestamp is not proof that the review followed the hold. Counting comments as a demand round or creating a second ledger would repair a different layer.

## Related

neomjs/neo-agent-skills#170 · #981 (distinct validator ownership).

Triage: promote the independently reproduced reader-contract violation found during review; I own its repair.
Sweeps: latest 20 open Brain issues and recent all-state A2A claims re-read immediately before filing; no equivalent. Helper independently inspected latest 50 plus exact-name/timestamp searches at 00:25:32Z; nearest #981 explicitly excludes this reader. KB returned no matching timestamp case. MC symptom sweep returned unrelated review handoffs; no prior ruling surfaced. Own-assignment sweep: seven open, none targets this reader. Structure map: existing `ai/services/github-workflow/shared/` helper, no new placement.

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b
Retrieval Hint: "merge hold disappears equal timestamp prior approval strictly newer clearance"

## Timeline

- 2026-10-11T00:31:16Z @neo-gpt assigned to @neo-gpt
- 2026-10-11T00:31:18Z @neo-gpt added the `bug` label
- 2026-10-11T00:31:18Z @neo-gpt added the `ai` label
- 2026-10-11T00:31:18Z @neo-gpt added the `agent-os` label
- 2026-10-11T00:34:47Z @neo-gpt cross-referenced by PR #170
- 2026-10-11T00:44:14Z @neo-gpt cross-referenced by PR #984
- 2026-10-11T01:18:15Z @tobiu referenced in commit `6b7e7d5` - "fix(github-workflow): preserve merge holds across tied reviews (#982) (#984)"
- 2026-10-11T01:18:15Z @tobiu closed this issue

