---
id: 525
title: Institution consumes the accepted Skills 0.1.29 correction
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt
createdAt: '2026-10-03T21:06:15Z'
updatedAt: '2026-10-04T01:07:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/525'
author: neo-gpt
commentsCount: 0
parentIssue: 140
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-04T01:07:36Z'
---
# Institution consumes the accepted Skills 0.1.29 correction

## Context
[Skills #140](https://github.com/neomjs/neo-agent-skills/issues/140) owns the delivery close of graduated [D#19384](https://github.com/orgs/neomjs/discussions/19384). Its consumer step explicitly covers Engine, Brain and Institution. Engine #19391 is approved; Emmy carries Brain #831. This ticket is the remaining planned Institution consumer, not an added feature.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-10-03T21:05Z; no equivalent. The complete live open PR/issue search and recent A2A claims distinguish Engine/Brain ownership and show no Institution consumer claim. KB suggested Engine #12577; live lookup shows that closed ticket governs CTAs, so it is not this work.

## The Problem
Institution dev `48178f7c49b0a418ec7b74ba09e81303de0920ce` still declares `neo-agent-skills@^0.1.24` and locks 0.1.24. Skills 0.1.29 carries the accepted correction. Source publication does not update this consumer.

## The Architectural Reality
The existing postinstall materializes ignored harness facades from the Skills package. Institution has no tracked root AGENTS carrier to regenerate. The two source authorities are package.json and package-lock.json; fresh recipient loading remains acceptance work in #140.

## The Fix
Raise the existing Skills minimum to `^0.1.29` and update its lock resolution to the published 0.1.29 artifact. Reuse the existing materializer and check command in an isolated worktree. Preserve unrelated dependency metadata and avoid creating another instruction carrier.

## Decision Record impact
`none`. Decision Record: Not needed. This executes the existing graduated consumer contract without introducing policy or architecture.

## Discussion Criteria Mapping
D#19384 R10 and Skills #140 step 2 → the Institution package/lock adoption. The fresh-session receipts and behavioral replays in #140 remain the outcome gate.

## Acceptance Criteria
- [ ] package.json declares `^0.1.29`; the lock resolves 0.1.29 with the published integrity.
- [ ] All unrelated dependency metadata is unchanged.
- [ ] An isolated installation resolves 0.1.29; existing materialization and its check pass; facade reads reach the accepted goal-first pickup and lesson-continuation text.
- [ ] The PR and #140 record exact source/package evidence while explicitly preserving fresh recipient load and replay acceptance as unvalidated.

## Out of Scope
FM features, Skills semantic changes, tracked generated AGENTS files, harness/service restarts and the shared replay close.

## Avoided Traps
Counting a pin as a live load receipt; modifying another maintainer's checkout; hand-editing materialized skills; resolving #140 from green CI alone; unrelated lock churn.

## Related
Refs neomjs/neo-agent-skills#140; neomjs/neo#19391; neomjs/neo-agent-brain#831.

Origin Session ID: 01a102a5-3799-7953-b73d-4238a2e1a210
Retrieval Hint: "Skills 140 Institution 0.1.29 planned consumer package lock materializer"


## Timeline

- 2026-10-03T21:06:16Z @neo-gpt added the `enhancement` label
- 2026-10-03T21:06:16Z @neo-gpt added the `agent-os` label
- 2026-10-03T21:06:17Z @neo-gpt added the `ai` label
- 2026-10-03T21:06:17Z @neo-gpt added the `build` label
- 2026-10-03T21:06:40Z @neo-gpt assigned to @neo-gpt
- 2026-10-03T21:06:43Z @neo-gpt added parent issue #140
- 2026-10-03T21:14:01Z @tobiu cross-referenced by PR #526
- 2026-10-03T21:15:42Z @neo-opus-vega cross-referenced by PR #831
- 2026-10-03T21:26:30Z @neo-gpt cross-referenced by #351
- 2026-10-04T01:07:36Z @tobiu closed this issue
- 2026-10-04T01:11:54Z @tobiu referenced in commit `4c65d45` - "fix(agentos): consume Skills 0.1.29 correction (#525) (#526)

Co-authored-by: neo-gpt <neo-gpt-euclid@neomjs.com>"

