---
id: 1108
title: 13 review-lint parity tests read neo's workflow from the Brain checkout
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T05:04:19Z'
updatedAt: '2026-10-18T05:25:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1108'
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
closedAt: '2026-10-18T05:25:31Z'
---
# 13 review-lint parity tests read neo's workflow from the Brain checkout

## Context

dev's Brain Unit base carries 48 red tests (the #201 census, comment 6245894428). 13 of them sit in `test/playwright/unit/ai/services/github-workflow/PullRequestService.spec.mjs`, and all 13 fail with one error: `ENOENT … neo-agent-brain/.github/workflows/agent-pr-review-body-lint.yml`.

## The Problem

The 13 tests hold neo's review-body lint workflow (its inline GitHub Script) and the Brain's `PullRequestService.validatePrReviewBody` to one contract, through shared fixtures. Examples are `#380: workflow lint reads the Round-1 Review ID like the service does` and the #13910, #15257 and #16148 arms.

`getAgentPrReviewBodyLintScript()` resolves the workflow from `process.cwd()`. But the workflow lives in neomjs/neo, and the path has no history in the Brain. The spec came over with the Brain split (neomjs/neo#17806). Since then these parity arms have not run anywhere, because neo has no test that reads its own workflow.

## The Architectural Reality

- `getAgentPrReviewBodyLintScript` (`PullRequestService.spec.mjs:1728`) reads `.github/workflows/agent-pr-review-body-lint.yml` relative to the cwd.
- The Brain installs the engine from its lock's git pin, which carries the repository tree, so `node_modules/neo.mjs/.github/workflows/agent-pr-review-body-lint.yml` exists.
- `neo.mjs` declares no `exports`, so `import.meta.resolve('neo.mjs/…')` reaches that file. The same spec already resolves `neo.mjs/src/Neo.mjs` this way.

## The Fix

Read the workflow through `import.meta.resolve('neo.mjs/.github/workflows/agent-pr-review-body-lint.yml')`. A pin move then carries the workflow with it.

Decision Record impact: none

## Acceptance Criteria

- [ ] **AC-1:** the 13 tests run and pass at head against the installed engine's workflow, and `PullRequestService.spec.mjs` passes whole.
- [ ] **AC-2:** the workflow is read from the installed engine, never from the Brain checkout's own tree.

## Out of Scope

- **Parity against neo's `dev` workflow instead of the pinned engine's.** The pin is the Brain's sync point, and a pin move re-runs the parity.
- **A workflow test inside neomjs/neo itself.**
- **Moving the lint into the shared PR baseline** (neomjs/neo-agent-skills#14, #22). If it moves, this read follows it.

## Related

- #201: the burn-down.
- #650.
- neomjs/neo#17806: the split.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T05:03:39Z; no equivalent found. `gh search` for the workflow name found only governance tickets (neomjs/neo-agent-skills#14, #22).
A2A claim sweep: the last 12 messages, all read-states; no claim on this spec. #1103, the github-workflow B4 conversion, does not touch it.
MC sweep: "PullRequestService spec review-body lint workflow file missing in the Brain checkout", no prior decision.
Own-assignment sweep: #201, which this ticket serves.
Structure map: N/A; the change is test-only, in the spec's own file.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "review-body lint parity tests ENOENT agent-pr-review-body-lint.yml Brain checkout"

## Timeline

- 2026-10-18T05:04:19Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T05:04:21Z @neo-opus-ada added the `bug` label
- 2026-10-18T05:04:21Z @neo-opus-ada added the `ai` label
- 2026-10-18T05:04:21Z @neo-opus-ada added the `testing` label
- 2026-10-18T05:05:04Z @neo-opus-ada cross-referenced by PR #1109
- 2026-10-18T05:25:31Z @tobiu referenced in commit `623a010` - "fix(test): the review-lint parity arms read the engine's workflow (#1108) (#1109)

getAgentPrReviewBodyLintScript resolved agent-pr-review-body-lint.yml from the Brain checkout, which never held it; it now reads it from the installed engine through import.meta.resolve, so the 13 arms that hold the workflow and validatePrReviewBody to one contract run again."
- 2026-10-18T05:25:31Z @tobiu closed this issue
- 2026-10-18T05:26:33Z @neo-opus-ada cross-referenced by #1110
- 2026-10-18T05:27:11Z @neo-opus-ada cross-referenced by PR #1111
- 2026-10-18T05:29:03Z @neo-opus-ada cross-referenced by #1112
- 2026-10-18T05:29:29Z @neo-opus-ada cross-referenced by PR #1113
- 2026-10-18T05:32:05Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T06:40:41Z @neo-opus-ada cross-referenced by #1117
- 2026-10-18T06:44:52Z @neo-opus-ada cross-referenced by PR #1118
- 2026-10-18T06:55:07Z @neo-opus-ada cross-referenced by #1119
- 2026-10-18T06:58:07Z @neo-opus-ada cross-referenced by PR #1120
- 2026-10-18T07:00:34Z @neo-opus-ada cross-referenced by #1121
- 2026-10-18T07:03:12Z @neo-opus-ada cross-referenced by PR #1122
- 2026-10-18T07:20:59Z @neo-opus-ada cross-referenced by #1125
- 2026-10-18T07:24:07Z @neo-opus-ada cross-referenced by PR #1126

