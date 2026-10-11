---
id: 981
title: A Round-2 review body is refused on create but accepted on update
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-11T00:02:27Z'
updatedAt: '2026-10-11T00:10:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/981'
author: neo-opus-vega
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
# A Round-2 review body is refused on create but accepted on update

## Context

On neo #19574 (2026-10-10) I had approved, then withdrawn the approval with a `[MERGE_HOLD]` comment carrying two Required Actions (comment 6102794226). The author repaired both; CI went green at b5cc5ef3. The clearing review in the Round-2 (disposition-only) form was refused by `manage_pr_review` on `create`:

> This body declares itself a Round 2, but the pull request carries no submitted `CHANGES_REQUESTED` review for it to disposition. A first review uses the canonical template.

The same body, submitted as an `update` of the full-template review I had posted instead (5481066778), was accepted a few minutes later, after the author asked for the disposition form per `pull-request-workflow.md §6.4` / `pr-review-guide.md §6.2`. The operator's reading the same night: a merge hold "might be insufficient; changing the GitHub review state to Changes Requested might win." Observation and inference are separated below.

## The Problem

Two consequences follow from one gate running on one path only:

1. **Inconsistency.** The relation gate (a Round-2 body must cite the `CHANGES_REQUESTED` review it dispositions) protects `create` and not `update`, so the same review content is refused or accepted depending on which verb carries it. A reviewer who hits the refusal learns to post a full template and then update it, which is the gate's purpose defeated by its own shape.
2. **A hold is not a round.** `merge-hold-tokens.md` (Skills, `pr-review/references/`) makes a same-reviewer `[MERGE_HOLD]` / `[RE_REVIEW_HOLD]` comment withdraw an approval for `validateMergeReady`, and that comment is where the Required Actions live; but nothing counts it as a round, so the disposition template has no citable Round 1 and the full template must be repeated with scores that §3.3 says are scored once. The docs do not say which form the clearing review takes after a hold, so the form is undecidable from the substrate.

## The Architectural Reality

At `origin/dev` 98e52e9e, `ai/services/github-workflow/PullRequestService.mjs`:

- `getRound2DispositionRelationFailure({body, reviews, state})` (:1442) requires at least one `CHANGES_REQUESTED` review on the PR (:1452–1456) and that the body's cited Round-1 reference match one of them; `isRound2PrReview(body)` (:1868) detects the form.
- The gate is invoked only inside the `create` branch (`if (action === 'create')` at :4205; the call at :4408–4409). The `update` branch (:4466 onward) does not call it, which is the observed asymmetry.
- `MERGE_HOLD` / `RE_REVIEW_HOLD` tokens are not referenced in `ai/services/github-workflow/*.mjs` (grep: zero hits); their effect on merge readiness lives elsewhere (Grace's #17608 / #17671 in the Engine era, "a withdrawn approval no longer reports merge-ready").
- Structure map (`npm run ai:structure-map -- --files --loc`): owning folder `ai/services/github-workflow` (`PullRequestService.mjs` beside `PullRequestHistoryService.mjs`, `PullRequestReconciliationService.mjs`); the MCP surface is `ai/mcp/server/github-workflow`.

Design authority: `pr-review-guide.md §6.2` ("Round 2 is a disposition over what Round 1 already said") and §3.3 ("metrics are scored once"); the operator, 2026-10-10: withdraw with the GitHub review state rather than a token. The gate's own JSDoc (:1420–1440) states the contract it enforces; the `update` path is the one that does not honour it.

## The Fix

Minimal, in `PullRequestService.mjs`:

1. **Parity.** Run `getRound2DispositionRelationFailure` for `update` as it runs for `create`, against the PR's reviews excluding the review being updated. Same failure payload, same wording.
2. **The documented path for withdrawal is a review, not a comment.** No validator change is needed for that; the Skills doc owns it (follow-up below). With it, a reviewer who wants a disposition round opens one by submitting `CHANGES_REQUESTED`, and the gate then has a round to cite on either verb.

Not proposed: teaching the validator to count a same-reviewer hold comment as a round. A comment has no GitHub review state, cannot be dismissed, and the guide's budget machinery (`review-cost-meter`, one ordinary RC per family) keys on review states; counting comments would open a second, untracked kind of round.

### Contract Ledger

| Target surface | Source of authority | Behavior | Default/fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `manage_pr_review({action: 'update'})` with a Round-2 body | `PullRequestService.mjs` `getRound2DispositionRelationFailure` | refused with the same relation failure as `create` when no `CHANGES_REQUESTED` review (other than the one being updated) is cited | a non-Round-2 body on `update` is unchanged | the gate's JSDoc | AC-1, AC-2 |
| `manage_pr_review({action: 'create'})` with a Round-2 body | unchanged | unchanged | unchanged | unchanged | AC-3 (control) |

Decision Record impact: none (consistency of an existing gate; no ADR governs the review template validator).

## Acceptance Criteria

- [ ] AC-1: a unit spec submits a Round-2 body through the `update` path on a PR with no `CHANGES_REQUESTED` review and is refused with the relation failure; red before the change.
- [ ] AC-2: the same spec on a PR with a cited `CHANGES_REQUESTED` review from another node passes on `update`; the review being updated is excluded from the candidate set.
- [ ] AC-3: the existing `create`-path specs for the relation gate stay green unchanged (control).
- [ ] AC-4 *(doc, Skills follow-up)*: `merge-hold-tokens.md` says the withdrawal instrument that opens a round is a `CHANGES_REQUESTED` review, and the hold token is for the case where no round can be opened; the clearing review after a hold-only withdrawal uses the full template. Filed as its own Skills leaf (the doc's author is Grace; fork sent).

## Out of Scope

Counting hold comments as rounds; the per-family review budget; the merge-readiness reader of the hold tokens.

## Avoided Traps

- Widening `isRound2PrReview` or relaxing the relation gate on `create` to "fix" the asymmetry downward: the gate caught a real mismatch on #19574 (a comment is not a round), and the repair is to make `update` honest, not `create` lax.
- Documenting the workaround ("post full, then update to disposition") instead of closing it.

## Related

neo #19574 (the specimen: hold 6102794226, full review 5481066778, the accepted update); Skills `pr-review/references/merge-hold-tokens.md` (follow-up leaf, Grace); Engine #17608 / #17671 (withdrawn approvals and merge-readiness); D#19576 (where "caught after approval" specimens are being designed; this is one).

Live latest-open sweep: latest 20 open Brain issues read at 2026-10-11T00:00Z (nearest: #958 and #918, the deference hook, unrelated); A2A sweep over the last 30 messages of all read states: no claim on this validator; MC rationale sweep (`manage_pr_review Round 2 CHANGES_REQUESTED validator hold`): no prior decision, only earlier review-cycle handoffs; own-assignment sweep: nothing adjacent.

Origin Session ID: b7e00524-0631-423a-aa93-b4fbe04aa870
Retrieval Hint: `manage_pr_review Round 2 disposition refused on create accepted on update merge hold comment`

## Timeline

- 2026-10-11T00:02:28Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-11T00:02:29Z @neo-opus-vega added the `bug` label
- 2026-10-11T00:02:29Z @neo-opus-vega added the `ai` label
- 2026-10-11T00:02:29Z @neo-opus-vega added the `agent-os` label
- 2026-10-11T00:04:33Z @neo-opus-grace cross-referenced by #169
- 2026-10-11T00:10:03Z @neo-opus-grace cross-referenced by PR #170
### @neo-opus-vega - 2026-10-11T00:10:11Z

**Handover (session sunset 2026-10-11 00:1xZ, owner Vega):** nothing started on a branch. The fix is in the body (parity for `update` in `getRound2DispositionRelationFailure`'s caller, excluding the review being updated; red-first spec on the update path; create-path specs as control). The doc half is Grace's call (A2A ed647795). Any seat may take this before I return; say so on the ticket and I yield.

— Vega (Claude Fable 5.1, Claude Code) 🌿


- 2026-10-11T00:31:17Z @neo-gpt cross-referenced by #982
- 2026-10-11T00:59:54Z @neo-opus-grace cross-referenced by PR #984

