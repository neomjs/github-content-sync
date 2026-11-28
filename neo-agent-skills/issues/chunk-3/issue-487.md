---
id: 487
title: Show reviewers how to score the approved head for Brain ingestion
state: CLOSED
labels:
  - documentation
  - ai
  - needs-design-authority
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-27T20:41:10Z'
updatedAt: '2026-11-27T21:21:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/487'
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
closedAt: '2026-11-27T21:21:55Z'
---
# Show reviewers how to score the approved head for Brain ingestion

## Context
Review scores are inputs to the Neo Brain’s graph-ingestion workflow. On Engine PR #22352 I first supplied prose under the metric tags and then copied the Round-1 numbers into the approval after the original defects had been repaired. The latter loses the distinction between the submitted and delivered work even if every number is syntactically valid. [Affected approval](https://github.com/neomjs/neo/pull/22352#pullrequestreview-6093103048)

## The Problem
The released skill already defines each score, requires seven post-fix values and validates their grammar. Its approval template lists fields and exceptions, while the instruction beside those fields leaves the reviewer to infer why the initial scores need reconsideration. A table with the score in a separate cell from the tag can also look right yet fail the existing parser.

## The Architectural Reality
Skills #480 and PR #483 own the current semantics. `pr-review-guide.md` §§3.1–3.3 own the rubric and its decile anchors. Brain #1949 and the published Skills validator own value syntax. The observed gap is author guidance at final approval. The installed Engine 2.10.0 copy also predates #483.

## The Fix
Show a self-contained, parser-readable final table in the existing approval and exceptional follow-up templates. Explain at the point of use that the published values describe the reviewed head for graph ingestion. Use the existing rubric to reconsider deductions against the repaired behavior. A still-earned score can stay the same. Retain the light-review and open-action exceptions.

## Acceptance Criteria
- [ ] AC-1 A reviewer following the guide and the two existing templates can produce a standalone final assessment of the whole PR with six scored reasons and its categorical effort profile, referenced to the existing rubric.
- [ ] AC-2 A filled table passes the released value parser and a split tag/value negative control fails. Preserve the existing Comment and Micro-Review paths, and synchronize the distributed plugin.
 
Owner: @neo-gpt-emmy. This refines the presentation shipped in #483. It introduces no new scoring scale or graph writer.
 
Origin Session ID: 197faabf-5fbb-4787-b578-7566483a7888
 
Retrieval Hint: final evaluation metrics table approved head graph ingestion decile anchors

## Timeline

- 2026-11-27T20:41:11Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-27T20:41:12Z @neo-gpt-emmy added the `documentation` label
- 2026-11-27T20:41:12Z @neo-gpt-emmy added the `ai` label
- 2026-11-27T20:41:28Z @github-actions added the `needs-design-authority` label
- 2026-11-27T20:46:56Z @neo-gpt-emmy cross-referenced by PR #488
- 2026-11-27T21:21:55Z @tobiu referenced in commit `d04c984` - "docs(pr-review): show the approved head's scores for Brain ingestion (#487) (#488)"
- 2026-11-27T21:21:56Z @tobiu closed this issue

