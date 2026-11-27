---
id: 1949
title: Validate and prefer final evaluation metrics on Round-2 Approved reviews
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-27T02:41:49Z'
updatedAt: '2026-11-27T11:20:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1949'
author: neo-gemini-maryam
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
closedAt: '2026-11-27T11:20:12Z'
---
# Validate and prefer final evaluation metrics on Round-2 Approved reviews

Design authority: `@tobiu` 2026-11-27 `Friction -> Gold` graph-ingestion finding (iterated PRs retain Round-1 `CHANGES_REQUESTED` evaluation scores because Round-2 `APPROVED` reviews omit metrics), `ai/services/github-workflow/PullRequestService.mjs` (`L1298–1306`, `L2227–2293`, `L4206–4260`), and `ai/services/ingestion/ConceptDiscoveryService.mjs`.

unowned-rationale: Maintainer self-select in lockstep with companion Skills ticket `neomjs/neo-agent-skills#480` and `neomjs/neo#22305`.

## Context

In `neomjs/neo-agent-brain`, `PullRequestService` (`manage_pr_review` and `validate_pr_review_body`) gates PR review bodies before submission, and downstream ingestion / evaluation readers consume the **6 numeric `0–100` scores** (`[ARCH_ALIGNMENT]`, `[CONTENT_COMPLETENESS]`, `[EXECUTION_QUALITY]`, `[PRODUCTIVITY]`, `[IMPACT]`, `[COMPLEXITY]`) plus **1 categorical `[EFFORT_PROFILE]`** (`Quick Win | Heavy Lift | Maintenance | Architectural Pillar`). Today, `getRound2PrReviewTemplateValidationFailure()` (`PullRequestService.mjs:2227–2293`) admits Round-2 `APPROVED` reviews with zero evaluation metrics — so any PR that went `Round 1 Request Changes` → author fix → `Round 2 Approved` has only its pre-fix `CHANGES_REQUESTED` metrics recorded on the PR.

## The Problem

1. **Pre-submit & post-submit validator parity + missing Round-2 `APPROVED` metrics:**
   - Once `neomjs/neo-agent-skills#480` requires `### 📊 Final Evaluation Metrics` on Round-2 `Approved` / `Approve+Follow-Up` reviews, `PullRequestService` must enforce the exact same grammar (`6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]`, or `N/A — Cycle 1 was Micro-Review` per `neomjs/neo-agent-skills#64`) so `manage_pr_review` and `validate_pr_review_body` agree 100% with `check-pr-review-body.mjs` (`neomjs/neo-agent-brain#1492` parity invariant).
2. **Latest-Approval metric precedence in graph / evaluation consumers:**
   - Verify whether any Brain evaluation or ingestion reader aggregates evaluation metrics across a PR's reviews; if none exists in Brain code, enforcing final metrics on the Round-2 `APPROVED` review text via `neomjs/neo-agent-skills#480` + `#1952` resolves the precedence record at the source.

## The Architectural Reality

- **Exact 7-tag value grammar (`D#22305#discussioncomment-19939331` Check 2):**
  - **6 numeric `0–100` scores:** `[ARCH_ALIGNMENT]`, `[CONTENT_COMPLETENESS]`, `[EXECUTION_QUALITY]`, `[PRODUCTIVITY]`, `[IMPACT]`, `[COMPLEXITY]`.
  - **1 categorical `[EFFORT_PROFILE]`:** `Quick Win | Heavy Lift | Maintenance | Architectural Pillar` (never treated as numeric!).
- `PullRequestService.mjs` inspects Round-2 `**Status:**` (`L4157`) and fetches prior PR reviews in `#validateRound2PrReviewCarry()` (`L4206+`):
  - Cited Round 1 was full form + Round 2 `Status` is `Approved` / `Approve+Follow-Up` → require all 7 evaluation-metric tags (`6` numeric `0–100` + `1` categorical `[EFFORT_PROFILE]`).
  - Cited Round 1 was `Micro-Review` + Round 2 `Status` is `Approved` → accept `N/A — Cycle 1 was Micro-Review` (and refuse a fake `N/A — Cycle 1 was Micro-Review` claim if Round 1 was full form).
  - Round 2 `Status` is `Comment` (`STILL_OPEN`) → metrics not required.
- **Brain reader audit (`git grep` across `ai/`, verified in `#1952` Round 1/2):** Outside `PullRequestService.mjs`, the 7 evaluation-metric tags appear only as prompt prose in `ai/services/ingestion/ConceptDiscoveryService.mjs` (`L33`, `L163`, `L506`); other Brain review readers (`focusV1Section`, `issueFocusSections`, `review-cost-meter`) read distinct fields, so zero Brain code readers parse evaluation-metric tags directly.

## The Fix

1. **Sync `PullRequestService.mjs` validation with `neomjs/neo-agent-skills#480`:**
   - Update `getRound2PrReviewTemplateValidationFailure()`, `getCanonicalPrReviewTemplateValidationFailure()`, and `#validateRound2PrReviewCarry()` in `ai/services/github-workflow/PullRequestService.mjs` to enforce `6` numeric `0–100` integer scores + `1` categorical `[EFFORT_PROFILE]` on Round-2 `Approved` / `Approve+Follow-Up` and canonical/follow-up reviews (or verified `N/A — Cycle 1 was Micro-Review` when `pr_number` is supplied), running the identical full (`Maintenance` profile), micro, and `Comment` test fixtures as `neomjs/neo-agent-skills#480`.
2. **Verify Brain review-metric readers:**
   - Confirm via `git grep` that no Brain code reader aggregates evaluation-metric tags outside `PullRequestService.mjs`, so carrying final metrics on Round-2 `APPROVED` reviews via `neomjs/neo-agent-skills#483` + `#1952` resolves AC-3 without dead service helpers.
3. **Unit test parity in `PullRequestService.spec.mjs`** covering full (`Maintenance` profile), `Micro-Review` `N/A` (valid + spoofed), and `Comment` (`STILL_OPEN`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ai/services/github-workflow/PullRequestService.mjs` (`validatePrReviewBody`, `managePrReview`) | `neomjs/neo-agent-skills#480`, `#34`, `#1492`, `D#22305` | Requires `6` numeric `0–100` integer scores + `1` categorical `[EFFORT_PROFILE]` on Round-2 `Approved` / `Approve+Follow-Up` and canonical/follow-up reviews (or verified `N/A — Cycle 1 was Micro-Review`); keeps `Status: Comment` exempt and `- [ ]` checklists forbidden | Legacy historical Round-2 approvals remain untouched on GitHub | JSDoc on `PullRequestService.mjs` | `PullRequestService.spec.mjs` (shared fixtures with `skills#480`) |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `PullRequestService#validatePrReviewBody` and `PullRequestService#managePrReview` enforce `6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]` (`Quick Win | Heavy Lift | Maintenance | Architectural Pillar`) on Round-2 `Approved` / `Approve+Follow-Up` reviews in lockstep with `neomjs/neo-agent-skills#480` (verified on shared `Maintenance`, `Micro-Review`, and `Comment` fixtures).
- [ ] `#validateRound2PrReviewCarry` verifies `N/A — Cycle 1 was Micro-Review` against the cited Round-1 review body (passes when Cycle 1 was `PR Micro-Review`, fails when Cycle 1 was full-form).
- [ ] Effective PR evaluation-metric resolution prefers post-fix Round-2 `APPROVED` metrics (`6` numeric + `1` categorical) over pre-fix Round-1 `CHANGES_REQUESTED` metrics, with legacy fallback for historical approvals — **verified with zero Brain code readers (`git grep` confirms metric tags appear outside `PullRequestService.mjs` only as prose in `ConceptDiscoveryService.mjs:33,163,506`, so the Round-2 `APPROVED` review text enforced by `#1952` + `neomjs/neo-agent-skills#483` carries the final metrics directly)**.
- [ ] Unit specs in `PullRequestService.spec.mjs` pass.

## Out of Scope

- Modifying `pr-review-round-2-template.md` and `check-pr-review-body.mjs` (`neomjs/neo-agent-skills#480`).

## Avoided Traps

- **Treating `[EFFORT_PROFILE]` as numeric (`D#22305#discussioncomment-19939331` Check 2):** Enforce `6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]`.
- **Desynchronizing `PullRequestService` from `check-pr-review-body.mjs` (`#1492`):** Share identical fixture expectations across both PRs before enabling.

## Related

- Companion `neomjs/neo-agent-skills` ticket: `neomjs/neo-agent-skills#480` (and `neomjs/neo#22305`).
- Related Brain tickets: #34 and #1492 (closed).

Structure-map gate (§1c): `ai/services/github-workflow/PullRequestService.mjs` (`5,056` LOC) verified directly in `../neo-agent-brain`.
Live latest-open + closed sweep: checked latest 25 open (`154` open at creation in `neomjs/neo-agent-brain`, red band; read `#34`, `#167`, `#1492`) at `2026-11-27T02:38Z`; updated with `D#22305` `STEP_BACK` Check 2 at `2026-11-27T02:58Z` and `#1952` Round-1 AC-3 no-reader audit at `2026-11-27T10:53Z`.

Origin Session ID: 23bade35-02da-4666-a059-3305968bb749

Retrieval Hint: "PullRequestService Round 2 Approved final evaluation metrics 6 numeric 1 categorical EFFORT_PROFILE RLAIF graph ingestion"

## Timeline

- 2026-11-27T02:41:51Z @neo-gemini-maryam added the `bug` label
- 2026-11-27T02:41:51Z @neo-gemini-maryam added the `ai` label
- 2026-11-27T02:41:51Z @neo-gemini-maryam added the `architecture` label
- 2026-11-27T02:41:51Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-27T02:41:52Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-27T04:59:35Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-27T05:17:21Z @neo-gemini-maryam cross-referenced by PR #1952
- 2026-11-27T05:27:04Z @neo-opus-vega cross-referenced by PR #483
- 2026-11-27T05:52:52Z @neo-gemini-maryam cross-referenced by PR #484
- 2026-11-27T10:52:52Z @neo-gemini-maryam referenced in commit `2b9ff55` - "fix(review): drop unconsumed evaluation-metric resolver and dead anchor list (#1949)"
- 2026-11-27T11:20:12Z @tobiu referenced in commit `debe6f2` - "fix(review): validate and prefer final evaluation metrics on Round-2 approvals (#1949) (#1952)

* fix(review): validate and prefer final evaluation metrics on Round-2 approvals (#1949)

* fix(review): drop unconsumed evaluation-metric resolver and dead anchor list (#1949)"
- 2026-11-27T11:20:13Z @tobiu closed this issue

