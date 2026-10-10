---
id: 970
title: Dependabot pull requests merge themselves once the required checks pass
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-10-10T16:32:39Z'
updatedAt: '2026-10-10T18:43:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/970'
author: neo-opus-grace
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
closedAt: '2026-10-10T18:43:36Z'
---
# Dependabot pull requests merge themselves once the required checks pass

## Context

Design authority: the operator, 2026-10-10 ~16:00Z: *"dependabot: ALL PRs once CI is green. no limitations."* neo-agent-skills 0.1.35 (neomjs/neo-agent-skills#160, merged) ships the reusable workflow. Each repository opts in with a caller, as it does for the PR baseline.

## The Problem

This repository's Dependabot pull requests wait for a human merge. Nothing here calls `reusable-dependabot-automerge.yml`.

## The Architectural Reality

- *Allow auto-merge* is on, and `dev`'s rulesets require 15 checks: the ten `Shared PR Baseline / …` jobs plus `unit`, `lint`, `substrate`, `integration-unified` and `integration-parity` (live read 2026-10-10). Product CI is required, so a green merge means the tests passed.
- The caller pattern: `.github/workflows/shared-pr-baseline.yml` names the Skills release by tag.

## The Fix

`.github/workflows/dependabot-automerge.yml`:
- `on: pull_request` (opened, synchronize, reopened);
- `permissions: contents: write, pull-requests: write`;
- one job, `if: github.event.pull_request.user.login == 'dependabot[bot]'`, `uses: neomjs/neo-agent-skills/.github/workflows/reusable-dependabot-automerge.yml@v0.1.35`.

## Acceptance Criteria

- [ ] AC-1 The caller exists as above. On its own PR, which is not Dependabot's, the job is skipped.
- [ ] AC-2 **Post-merge:** the next Dependabot pull request merges itself once the required checks pass; its link is the receipt.

## Out of Scope

The ruleset's required checks (the operator's setting); `dependabot.yml`'s grouping.

## Related

neomjs/neo-agent-skills#14 (epic), neomjs/neo-agent-skills#159.

Sweeps: live latest-open sweep of this repository's 20 newest open issues at 16:31Z plus an org search for "dependabot auto-merge", no equivalent · A2A: my Skills #159 lane-claim names this rollout · MC: the 00:24Z and 16:00Z decisions (Skills #154, #159) · Own-assignment: none overlapping.

Origin Session ID: 9ea1c6d0-80b1-4552-b07d-004bb1dba240
Retrieval Hint: "dependabot auto-merge caller reusable-dependabot-automerge v0.1.35 brain"

## Timeline

- 2026-10-10T16:32:40Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-10T16:32:41Z @neo-opus-grace added the `enhancement` label
- 2026-10-10T16:32:41Z @neo-opus-grace added the `ai` label
- 2026-10-10T16:32:41Z @neo-opus-grace added the `build` label
- 2026-10-10T16:33:52Z @neo-opus-grace referenced in commit `032c238` - "ci(dependabot): Dependabot pull requests merge themselves on green required checks (#970)"
- 2026-10-10T16:33:54Z @neo-opus-grace cross-referenced by PR #972
- 2026-10-10T18:43:36Z @tobiu referenced in commit `69120d2` - "ci(dependabot): Dependabot pull requests merge themselves on green required checks (#970) (#972)"
- 2026-10-10T18:43:37Z @tobiu closed this issue

