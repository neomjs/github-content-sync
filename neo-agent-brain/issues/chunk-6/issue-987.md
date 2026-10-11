---
id: 987
title: The deference Stop hook's unit suites did not survive the Engine→Brain move
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - regression
assignees:
  - neo-opus-ada
createdAt: '2026-10-11T01:34:57Z'
updatedAt: '2026-10-11T01:35:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/987'
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
---
# The deference Stop hook's unit suites did not survive the Engine→Brain move

## Context

Engine commit `c623b2f63c` ("remove received Brain implementation", neomjs/neo#17806) deleted two Stop-hook suites together with the Brain code they covered:
- `test/playwright/unit/hooks/deferencePhraseMatch.spec.mjs`: 303 lines, 23 tests;
- `test/playwright/unit/hooks/stopHookDecision.spec.mjs`.

The sources live on in Brain (`ai/scripts/lifecycle/deferencePhraseMatch.mjs`, `stopHookDecision.mjs`). The specs exist in neither repository: I checked both `dev` trees on 2026-10-11.

- #250 restored and projected the hooks themselves. Its ACs covered `initServerConfigs.spec` and `bootstrapWorktree.spec`, not these two suites.
- #985 (for #918) adds a new `deferencePhraseMatch.spec.mjs` with 26 cases. The lost 23-test suite passes 23/23 against #985's head `910090286c` (run in review, its assertions unchanged).
- #78 still cites the lost `stopHookDecision.spec.mjs` as the place that pins the operator-dialogue boundary.

## The Problem

The matcher and decision layer of an enforcing Stop hook have had no regression net in Brain since the move. Edits of the #918 and #958 class, and #78's carve question, land against tests that no longer exist. The lost suite pinned the matcher's subtle arms:
- clause-terminal grammar against Markdown layout;
- the closed indefinite-object vocabulary;
- identifier-safe emphasis stripping;
- the citation bridge's allowlist;
- the reminder's routing text.

## The Fix

Port both suites from `c623b2f63c^` into Brain under `test/playwright/unit/ai/scripts/lifecycle/`, beside the sources. Merge the deference arms into the canonical spec that #985 creates, without duplicating its cases, and adjust the imports. For any arm that no longer holds, either restore the behavior or state in the PR which behavior moved and where.

## Acceptance Criteria

- [ ] The lost `deferencePhraseMatch` arms run in Brain CI, merged with #985's spec, with no duplicate cases.
- [ ] The lost `stopHookDecision` arms run in Brain CI, or each dropped arm names the behavior that moved.
- [ ] #78's evidence pointer names the restored spec.

## Out of Scope

- Any behavior change to the matcher or the decision layer.
- #958's attribution false positive.

## Related

neomjs/neo#17806 · #250 · #985 · #918 · #78 · #958

Sequencing: after #985 merges, since both PRs write the same spec file.

Sweeps:
- Live latest-open sweep: the latest 20 open Brain issues at 01:3xZ, plus searches for "stopHookDecision spec", "deferencePhraseMatch spec", "deference tests lost", "deference hook tests" and "received Brain implementation tests". Hits are #78, #124, #918 and #958; none covers the lost suites.
- A2A in-flight sweep: no claim.
- MC sweep: "deference hook unit spec stopHookDecision deferencePhraseMatch tests missing after Engine Brain move". It found Grace's 2026-08-30 diagnosis (filed as #250), whose scope is the hooks rather than these suites.
- Own-assignment sweep: none overlapping.

Origin Session ID: a6673f82-f995-4e40-83a9-3093d4ecd357
Retrieval Hint: "deference stop hook unit suites lost engine brain move c623b2f63c stopHookDecision deferencePhraseMatch"


## Timeline

- 2026-10-11T01:34:59Z @neo-opus-ada added the `enhancement` label
- 2026-10-11T01:34:59Z @neo-opus-ada added the `ai` label
- 2026-10-11T01:34:59Z @neo-opus-ada added the `testing` label
- 2026-10-11T01:34:59Z @neo-opus-ada added the `regression` label
- 2026-10-11T01:35:03Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-11T01:37:15Z @neo-opus-ada cross-referenced by PR #985

