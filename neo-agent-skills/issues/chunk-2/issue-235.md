---
id: 235
title: Run the reusable PR-review check on this repository's own reviews
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-10-29T02:30:16Z'
updatedAt: '2026-10-29T12:00:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/235'
author: neo-opus-vega
commentsCount: 1
parentIssue: 14
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-29T12:00:58Z'
---
# Run the reusable PR-review check on this repository's own reviews

## Context

#22 (PR #233, released as 0.1.73) published `.github/workflows/reusable-pr-review.yml` and the `neo-agent-skills/pr-review-policy` it runs. No repository calls the workflow yet. This repository is where the motivating review landed: review `5683341608` on #232 scored its metrics in stars and carried none of the template's tags, and nothing flagged it, because Skills runs no review lint. #233's second Post-Merge Validation item waits on a first live caller.

## The Problem

- `skill-corpus.yml` fires on `pull_request`, `push` and `issues`. Nothing fires on `pull_request_review`, so an agent review posted through `gh pr review` or the UI is never judged here.
- The reusable workflow has unit, mutation and mocked-script coverage, but no live run. Three of its properties only show on a real review event:
  - the caller grant reaching both jobs;
  - a failed `Review body` job's outputs reaching `Review correction`;
  - the comment upsert as `github-actions[bot]`.

## The Architectural Reality

- `reusable-pr-review.yml` at `v0.1.73`: `workflow_call` only. `Review body` is read-only (contents, issues and pull-requests read), and `Review correction` is the only writer (`pull-requests: write`). Its header prints the caller shape this ticket adds.
- Its `Review body` job refuses a caller that is not at the tag whose `package.json` it installs, so the caller must name `@v0.1.73`.
- For `pull_request_review`, GitHub runs the workflow from the pull request's merge ref, so the PR that adds the caller already exercises it on its own reviews.
- Today the shared baseline is called by neo (`pr-baseline.yml`), Brain and Institution (`shared-pr-baseline.yml`). neo-stage calls neither, and this repository runs its own `skill-corpus.yml`.

## The Fix

Add `.github/workflows/pr-review.yml`:

```yaml
name: PR review

on:
  pull_request_review:
    types: [submitted, edited]

permissions:
  contents: read
  issues: read
  pull-requests: write

jobs:
  review:
    uses: neomjs/neo-agent-skills/.github/workflows/reusable-pr-review.yml@v0.1.73
```

There is no `budget_activation_issue`. This repository was created on 2026-08-25, after the review budget activated (2026-07-16 by neo#15307), so every agent Request Changes here owes provenance, which is the workflow's default. The check stays informative: no ruleset change.

## Acceptance Criteria

- [ ] AC-1: The caller carries exactly the trigger, grant and tag above, and `check-workflow-concurrency` stays green.
- [ ] AC-2: A real agent review on the PR runs `review / Review body` against the live review, and a valid review passes.
- [ ] AC-3: A failing agent review turns `Review body` red and creates exactly one `Review correction` comment as `github-actions[bot]`. Editing the same review updates that comment rather than adding another. An author's COMMENT that opens a Required Actions item exercises this without approving or requesting changes.
- [ ] AC-4: #233's second Post-Merge Validation item is ticked with the run URLs.

## Out of Scope

- Callers in neo (which replaces its inline `agent-pr-review-body-lint.yml`), Brain, Institution and Stage. They follow once this live run passes, so a defect it finds lands in one repository's patch release rather than in five callers.
- Brain importing the policy into `PullRequestService`.
- Making the check required.

## Related

Parent #14 · source #22 / PR #233 · #80 (consumer coordinates; this adds one) · #144 (the Dependabot bump must move this pin with the baseline's)

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues on `neomjs/neo-agent-skills` at 2026-10-29T02:29Z; no equivalent. A2A sweep of the latest 30 messages: no claim on this scope. Memory Core sweep on the problem's nouns: no prior decision. Own open assignments (#144, #6, #5): none overlaps.

Owner: @neo-opus-vega, self-assigned at creation.

Origin Session ID: 7d196c25-ccd7-4cd8-9e70-094ee3597411

Retrieval Hint: "first live caller of reusable-pr-review.yml in neo-agent-skills"

## Timeline

- 2026-10-29T02:30:17Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-29T02:30:18Z @neo-opus-vega added the `enhancement` label
- 2026-10-29T02:30:19Z @neo-opus-vega added the `ai` label
- 2026-10-29T02:30:19Z @neo-opus-vega added the `build` label
- 2026-10-29T02:30:24Z @neo-opus-vega added parent issue #14
- 2026-10-29T02:30:28Z @github-actions added the `needs-design-authority` label
- 2026-10-29T02:31:58Z @neo-opus-vega cross-referenced by PR #236
- 2026-10-29T02:34:41Z @neo-opus-vega cross-referenced by PR #233
### @neo-opus-vega - 2026-10-29T02:35:54Z

**Session handover (2026-10-29 ~02:40Z, solo-refresh).** Still mine. PR #236 is open at `7350c7b`. AC-1 to AC-4 are met; the live runs are listed in the PR body.

Pickup:
1. Re-read `gh pr checks 236`; corpus was still running at 02:35Z. Once it is green, request one cross-family primary reviewer and send the matching waking DM. Maryam reviewed #233; Julia is the other Gemini seat.
2. After the merge, file the other caller leaves under #14, one per repository, and pin the then-current tag in each:
   - neo replaces its inline `agent-pr-review-body-lint.yml` with a caller passing `budget_activation_issue: 15257`. Only that leaf exercises the live activation-history read and its `issues: read` grant.
   - Brain, Institution and Stage each add a caller.
   - Brain imports `neo-agent-skills/pr-review-policy` into `PullRequestService`.
   - Each leaf notes #80/#144: the new pin must move with the baseline's.
3. Folded in as known: Grace's 10-28 defect-note says the micro-delta template wraps its origin placeholder in backticks, which `ORIGIN_SESSION` refuses when the line is filled in place. A small template-or-policy leaf; not yet filed.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-29T03:28:13Z @neo-opus-grace cross-referenced by #237
- 2026-10-29T12:00:58Z @tobiu referenced in commit `9399259` - "feat(skills): run the reusable PR-review check on this repository's own reviews (#235) (#236)

Skills calls reusable-pr-review.yml@v0.1.73 on pull_request_review, so an agent review posted around Brain's managed gate is judged here too. That is the gap through which #232's star-rated review went unflagged. This caller's own reviews are the workflow's first live run. Bumps 0.1.73 to 0.1.74, as every maintainer pull request carries a release."
- 2026-10-29T12:00:58Z @tobiu closed this issue
- 2026-10-29T13:27:44Z @neo-opus-vega cross-referenced by #20173

