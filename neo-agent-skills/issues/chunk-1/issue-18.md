---
id: 18
title: Make source-comment archaeology reusable across repositories
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - testing
  - build
  - model-experience
assignees:
  - neo-gpt
createdAt: '2026-08-30T00:44:18Z'
updatedAt: '2026-08-30T06:55:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/18'
author: neo-gpt
commentsCount: 0
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
closedAt: '2026-08-30T06:55:34Z'
---
# Make source-comment archaeology reusable across repositories

## Context

The Engine owns `buildScripts/util/check-ticket-archaeology.mjs` and `.github/workflows/ticket-archaeology-lint.yml`. The guard scans comments and JSDoc across every changed `.mjs` file and rejects decaying issue, PR, Discussion, Epic, ADR, and review-history references.

[Brain PR #242](https://github.com/neomjs/neo-agent-brain/pull/242) added ticket/review chronology across production source, tests, and CI while passing every Brain check. The Engine guard would also miss Brain-sized issue IDs because its current pattern assumes four or more digits.

This is a source leaf of #14. #15 and PR #17 established the reusable-workflow precedent.

## Problem

- The rule is absent outside Engine.
- Copying the implementation and workflow into every consumer would create drift.
- Issue numbering is repository-local, so `#\d{4,}` is not portable.
- An escape marker must not be able to legitimize an actual tracking reference.
- A green workflow that scans only added lines would leave archaeology elsewhere in the touched file.

## Intended architecture

The Skills package owns:

1. a portable comment-aware CLI under `scripts/`, published through the package bin;
2. a reusable `workflow_call` workflow with least-privilege permissions and a stable job name;
3. mutation-backed contract tests for the CLI and workflow.

The caller checks out its own exact revision and invokes the immutable Skills release. Consumer caller files remain separate leaves under #14.

By default, the CLI scans every changed tracked `.mjs` file in full, with explicit generated/projection exclusions. It distinguishes comments/JSDoc from string literals and recognizes repository-local IDs of any length plus named PR, issue, ticket, Epic, Discussion, ADR, and review-cycle archaeology.

## Contract ledger

| Surface | Authority |
|---|---|
| Detection behavior | Skills CLI and its tests |
| Reusable execution | Skills workflow |
| Required-check identity | Stable reusable job name |
| Distribution | Skills package bin |
| Consumer adoption | Separate repo-local callers under #14 |

No ADR change is required; this follows #14 and the reusable baseline established by #15 / PR #17.

## Acceptance criteria

- [ ] A Skills-owned CLI scans changed tracked `.mjs` files in full, not only added lines.
- [ ] Detection covers one-or-more-digit repository-local issue/PR/ticket references and named Epic, Discussion, ADR, and review-cycle archaeology.
- [ ] String literals and non-tracking numeric forms such as colors remain valid controls.
- [ ] An escape may cover a proven non-tracking false positive, but cannot hide a real tracking reference.
- [ ] Tests cover line, block, and JSDoc comments; 1-, 2-, 3-, and 5-digit references; pre-existing archaeology in a touched file; string and color controls; and red-control mutations.
- [ ] A least-privilege reusable `workflow_call` runs against the caller repository's exact diff and exposes a stable job name.
- [ ] Workflow contract tests prove caller checkout, immutable command/pin semantics, and actual guard execution.
- [ ] The published package contains the CLI; it does not rely on postinstall mutation of consumer workflows.
- [ ] This source PR does not edit consumer repositories.
- [ ] Parent #14 retains post-release consumer-adoption ownership; this source leaf neither edits nor pre-creates caller files.

## Out of scope

- Subjective prose-length, placement, or style linting.
- Product/runtime tests.
- Required-context settings.
- Consumer caller files in the source PR.
- Postinstall-written workflows.

## Avoided traps

- Copying the Engine workflow into each repository.
- Retaining the Engine-only four-digit ID assumption.
- Letting a suppression marker preserve actual ticket history.
- Scanning only added lines.
- Shipping a reusable workflow without caller adoption leaves.
- Turning intent-driven documentation judgment into a brittle style linter.

## Related

- Parent: #14
- Precedent: #15 and PR #17
- Engine residual: neomjs/neo#17783
- Triggering hold: neomjs/neo-agent-brain#242

## Origin

- Agent: @neo-gpt (Euclid)
- Session: `6243a442-7ced-4c0f-81c6-99b0f8358e63`
- Retrieval hint: `ticket archaeology reusable workflow repository-local IDs Brain PR 242`

## Creation notes

Live duplicate sweeps covered the Skills open queue, organization issue search, Knowledge Base, and recent A2A claims immediately before filing. No equivalent source leaf was found.


## Timeline

- 2026-08-30T00:44:19Z @neo-gpt added the `enhancement` label
- 2026-08-30T00:44:20Z @neo-gpt added the `ai` label
- 2026-08-30T00:44:20Z @neo-gpt added the `architecture` label
- 2026-08-30T00:44:20Z @neo-gpt added the `testing` label
- 2026-08-30T00:44:20Z @neo-gpt added the `build` label
- 2026-08-30T00:44:20Z @neo-gpt added the `model-experience` label
- 2026-08-30T00:44:25Z @neo-gpt added parent issue #14
- 2026-08-30T00:48:24Z @neo-gpt cross-referenced by PR #242
- 2026-08-30T00:50:23Z @neo-gpt assigned to @neo-gpt
- 2026-08-30T01:27:30Z @neo-gpt cross-referenced by PR #19
- 2026-08-30T02:07:08Z @neo-gpt cross-referenced by PR #17880
- 2026-08-30T02:17:24Z @tobiu referenced in commit `4412c5d` - "fix(ci): cover camelCase color properties (#18)"
- 2026-08-30T06:55:34Z @tobiu referenced in commit `d6551c8` - "Merge pull request #19 from neomjs/codex/18-reusable-ticket-archaeology

feat(ci): share source-comment archaeology guard (#18)"
- 2026-08-30T06:55:35Z @tobiu closed this issue
- 2026-08-30T17:53:59Z @neo-gpt-emmy cross-referenced by #22
- 2026-08-30T18:03:13Z @neo-gpt-emmy cross-referenced by #14
- 2026-09-04T12:05:36Z @neo-gpt cross-referenced by PR #18270
- 2026-09-07T00:14:47Z @neo-opus-grace cross-referenced by #56
- 2026-09-15T02:07:04Z @neo-opus-ada cross-referenced by #69

