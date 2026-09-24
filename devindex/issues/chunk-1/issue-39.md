---
id: 39
title: 'The shared PR baseline is called by a commit SHA, not its published release'
state: OPEN
labels:
  - enhancement
  - github_actions
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T20:30:25Z'
updatedAt: '2026-09-24T20:30:26Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/39'
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
---
# The shared PR baseline is called by a commit SHA, not its published release

## Context

The operator, 2026-09-24: *"we want npm versions. published skill releases. NEVER EVER SHA values. if there are SHA values, that must get changed."*

`.github/workflows/shared-pr-baseline.yml:17` calls the shared baseline by commit SHA: `neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml@72965ba56b43…`, which is Skills 0.1.2 from #11. `neo-agent-skills` now tags its publishes, and `v0.1.17` is on origin at npm 0.1.17's `gitHead` (`f7b29e7`).

## The Fix

Call the shared baseline at `@v0.1.17`. Dependabot's `github-actions` ecosystem, already enabled daily here, then moves the tag on each release.

The 0.1.17 baseline runs eight jobs where 0.1.2 ran two. The new ones are PR body, commit authorship, substrate size, source comment archaeology, npm overrides and secrets. #11 put the first two out of scope while 0.1.2 had neither. The caller needs two more changes:
- **Permissions:** grant `pull-requests: read` beside `contents: read`. The PR-body job reads the pull request itself. On neo's caller, a caller that granted less failed at startup with no jobs.
- **Trigger:** `types: [opened, edited, synchronize, reopened, ready_for_review]`, which the called workflow's header requires. With it, a corrected body re-runs the PR-body job without a push.

## Acceptance Criteria

- [ ] **AC-1:** `shared-pr-baseline.yml` calls `reusable-pr-baseline.yml@v0.1.17`, and no workflow here references `neo-agent-skills` by SHA.
- [ ] **AC-2:** The caller grants `contents: read` and `pull-requests: read`, and fires on `opened, edited, synchronize, reopened, ready_for_review`.
- [ ] **AC-3:** The introducing PR's own baseline run is green at the tag, across all eight jobs.

## Related

- **Parent:** neomjs/neo-agent-skills#114, consumers call by version.
- **Adoption at 0.1.2:** #11.
- **The other two callers:** neomjs/neo#19198 and neomjs/neo-agent-institution#190.

Sweeps at 20:31Z:
- **Latest open:** both open issues (#1, #9), neither about the baseline.
- **Search:** `SHA`, `baseline` and `neo-agent-skills` find only the closed #11, #15, #27 and #33.
- **A2A:** no claim.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T20:30:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T20:30:26Z @neo-opus-grace added the `enhancement` label
- 2026-09-24T20:30:26Z @neo-opus-grace added the `github_actions` label

