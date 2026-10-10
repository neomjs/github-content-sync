---
id: 161
title: Dependabot pull requests merge themselves once the required checks pass
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-10-10T16:32:53Z'
updatedAt: '2026-10-10T17:25:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/161'
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
closedAt: '2026-10-10T17:25:23Z'
---
# Dependabot pull requests merge themselves once the required checks pass

## Context

Design authority: the operator, 2026-10-10 ~16:00Z: *"dependabot: ALL PRs once CI is green. no limitations."* 0.1.35 (#160, merged) ships the reusable workflow. This repository calls its own published release like any consumer, so the code that decides a merge never comes from the pull request being merged.

## The Problem

This repository's own Dependabot pull requests (npm and github-actions) wait for a human merge. Nothing here calls `reusable-dependabot-automerge.yml`.

## The Architectural Reality

- *Allow auto-merge* is on, and `dev`'s ruleset requires `corpus` (live read 2026-10-10).
- A merge GitHub performs for this token starts no `push` runs, so an auto-merged Dependabot pull request publishes nothing; the README's "Dependabot merges do neither" stays true.
- The caller pins the published tag, never `./.github/workflows/…`: a local path would run the pull request's own copy.

## The Fix

`.github/workflows/dependabot-automerge.yml`:
- `on: pull_request` (opened, synchronize, reopened);
- `permissions: contents: write, pull-requests: write`;
- one job, `if: github.event.pull_request.user.login == 'dependabot[bot]'`, `uses: neomjs/neo-agent-skills/.github/workflows/reusable-dependabot-automerge.yml@v0.1.35`.

## Acceptance Criteria

- [ ] AC-1 The caller exists as above. On its own PR, which is not Dependabot's, the job is skipped.
- [ ] AC-2 **Post-merge:** the next Dependabot pull request here merges itself once `corpus` passes; its link is the receipt.

## Out of Scope

The ruleset's required checks; `dependabot.yml`'s grouping; the publish trigger.

## Related

#14 (epic), #159, #160.

Sweeps: live latest-open sweep of this repository's 20 newest open issues at 16:31Z plus an org search for "dependabot auto-merge", no equivalent · A2A: my #159 lane-claim names this rollout · MC: the 00:24Z and 16:00Z decisions (#154, #159) · Own-assignment: none overlapping.

Origin Session ID: 9ea1c6d0-80b1-4552-b07d-004bb1dba240
Retrieval Hint: "dependabot auto-merge caller reusable-dependabot-automerge v0.1.35 skills self"

## Timeline

- 2026-10-10T16:32:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-10T16:32:55Z @neo-opus-grace added the `enhancement` label
- 2026-10-10T16:32:55Z @neo-opus-grace added the `ai` label
- 2026-10-10T16:32:55Z @neo-opus-grace added the `build` label
- 2026-10-10T16:34:01Z @neo-opus-grace referenced in commit `2c61e34` - "ci(dependabot): Dependabot pull requests merge themselves on green required checks (#161)"
- 2026-10-10T16:34:04Z @neo-opus-grace cross-referenced by PR #162
- 2026-10-10T16:40:40Z @neo-opus-grace referenced in commit `830926e` - "chore(release): 0.1.36 carries the repository's own Dependabot caller (#161)"
- 2026-10-10T17:25:23Z @tobiu referenced in commit `02aa69b` - "ci(dependabot): Dependabot pull requests merge themselves on green required checks (#161) (#162)

* ci(dependabot): Dependabot pull requests merge themselves on green required checks (#161)

* chore(release): 0.1.36 carries the repository's own Dependabot caller (#161)"
- 2026-10-10T17:25:23Z @tobiu closed this issue

