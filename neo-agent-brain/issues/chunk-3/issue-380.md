---
id: 380
title: Round 2 refuses a linked Round-1 Review ID and admits an empty one
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-19T16:31:52Z'
updatedAt: '2026-09-19T18:18:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/380'
author: neo-opus-ada
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
closedAt: '2026-09-19T18:18:23Z'
---
# Round 2 refuses a linked Round-1 Review ID and admits an empty one

## Context

Two seats hit this on 2026-09-19. The managed Round 2 on neomjs/neo#18949 (@neo-gpt) and on neomjs/neo#18970 (@neo-gpt-emmy) were both refused with "names no Round-1 review to disposition" for a `[5256218531](URL)` value. Both reposted with a bare `PRR_…` id. It reached me as a receipt on the neighbouring Round-2 lane (#378). I measured the predicate at Brain `dev@77ee09e`.

## The Problem

`getRound2PrReviewTemplateValidationFailure()` tests `/\*\*Round-1 Review ID:\*\*\s*(?!\[)\S/` (`PullRequestService.mjs:2166`). The comment above it states the intent: the value must point somewhere, and the template's bracketed placeholder must not survive into a posted body. The test keys on the first character alone:

| Value after the anchor | Today | Intended |
|---|---|---|
| `[reviewId or URL] · …` (the template placeholder) | refused | refused |
| `[5256218531](https://…#pullrequestreview-5256218531)` | **refused** | accepted |
| `PRR_…`, `` `PRR_…` ``, `5107338867`, a bare URL, `PRR_… ([id](url))` | accepted | accepted |
| nothing, then ` · **Author Response:** …` | **accepted** | refused |
| nothing, then a newline: `\s*` crosses it and reads the next line's `*` | **accepted** | refused |

Every format in the accepted row occurs in real Round-2 bodies. The sample: PR reviews across neomjs/neo, neo-agent-brain and neo-agent-skills.

## The Architectural Reality

- Owner: `ai/services/github-workflow/PullRequestService.mjs`, `getRound2PrReviewTemplateValidationFailure()`. The check is body-shape only; proving the id names a real prior round is the caller's job (the comment at `:2146-2158`).
- The line it guards is Skills' `.agents/skills/pr-review/assets/pr-review-round-2-template.md:12`: `**Round-1 Review ID:** [reviewId or URL] · **Author Response:** [commentId or URL]`.
- Arms: `test/playwright/unit/ai/services/github-workflow/PullRequestService.spec.mjs`. Its existing fixtures use `PRR_123` and `PRR_prior`.
- Structure map: `ai/services/github-workflow` holds 15 files; the fix stays inside the owning function.
- A second copy: the Engine's post-submit review lint keeps this predicate byte for byte (`.github/workflows/agent-pr-review-body-lint.yml:433`). Its mirror is neomjs/neo#18988, and the two land together, or the managed path and CI disagree.

## The Fix

One predicate. The value must sit on the anchor's own line. It must not be a bracketed placeholder, meaning a `[…]` not followed by `(`. It must not start with the `·` separator:

`/\*\*Round-1 Review ID:\*\*[ \t]*(?!\[[^\]\n]*\](?!\())[^\s·]/`

Measured against the table, every row lands in its Intended column. The refusal message is unchanged.

## Acceptance Criteria

- [ ] A Markdown-link Round-1 Review ID is accepted.
- [ ] The template placeholder, an empty value before the separator, and an empty value at the end of the line are refused, with the existing message.
- [ ] Every format in the table's accepted row stays accepted.
- [ ] The arms for the three changed rows fail against dev's predicate.

## Out of Scope

- Resolving the id against the PR's live reviews.
- An `Author Response` check. None exists today, and nobody has reported missing it.
- Round-2 terminality (#34).

## Related

#34 · #378 · neomjs/neo#18988 (the CI copy) · neomjs/neo#18949 · neomjs/neo#18970

Live latest-open sweep: the latest 20 open issues at 2026-09-19T16:31Z, none equivalent. An all-states search for "Round-1 Review ID" finds #34 (terminality, not this predicate) and #378 (closed; micro Findings).
A2A sweep: latest 30, all read states; no claim on this predicate.
MC sweep: "Round 2 review rejected Round-1 Review ID markdown link placeholder validator refused", 6 results, no prior decision found.
Own-assignment sweep: 19 open, none on the Round-2 predicate.
Decision Record impact: none.

Origin Session ID: 6ecb7b5f-dc26-48a3-8e49-7232159377c1
Retrieval Hint: "Round-1 Review ID placeholder markdown link Round 2 refused"


## Timeline

- 2026-09-19T16:31:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-19T16:31:53Z @neo-opus-ada added the `bug` label
- 2026-09-19T16:31:53Z @neo-opus-ada added the `ai` label
- 2026-09-19T16:31:54Z @neo-opus-ada added the `agent-os` label
- 2026-09-19T16:36:27Z @neo-opus-ada cross-referenced by #18988
- 2026-09-19T16:38:48Z @neo-opus-ada cross-referenced by PR #381
- 2026-09-19T18:18:23Z @tobiu referenced in commit `407a895` - "Merge pull request #381 from neomjs/ada/380-round1-review-id

fix(github-workflow): a Round 2 may link its Round-1 Review ID, and an empty one is refused (#380)"
- 2026-09-19T18:18:23Z @tobiu closed this issue

