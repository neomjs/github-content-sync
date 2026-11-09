---
id: 896
title: Dependabot holds the Skills workflow refs three days after every release
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-11-08T22:27:44Z'
updatedAt: '2026-11-09T12:10:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/896'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-11-09T12:10:06Z'
---
# Dependabot holds the Skills workflow refs three days after every release

## Context

On 2026-11-08 the `github-actions` update (#893) moved this repository's `neomjs/neo-agent-skills` workflow refs only to `v0.1.90`. It skipped `0.1.91` through `1.2.0`, every one of them less than three days old. The npm update in the same run took the newest `neo-agent-skills`, because #530 exempted the package from Dependabot's default three-day cooldown. The `github-actions` entry has no `cooldown`, so the default hold applies to it. GitHub's docs support `default-days` and `exclude` for GitHub Actions. Grace measured this across all four consumers; the Engine's fix is neomjs/neo#20618 → neomjs/neo#20619.

## The Problem

Each Skills release reaches the workflows this repository calls three days after its npm package does. Until then, the installed skills and the shared workflows they run with sit on different releases.

## The Architectural Reality

- Design authority: n/a — the npm entry already states the policy ("Our own Skills releases skip Dependabot's three-day default hold"). This applies it to the second ecosystem.
- `.github/dependabot.yml`'s `github-actions` entry carries `schedule` and `groups` only.

## The Fix

The `github-actions` entry gets the block from neomjs/neo#20619, verbatim: `cooldown: {default-days: 3, exclude: ["neomjs/neo-agent-skills*"]}` and its two comment lines.

Decision Record impact: none.

## Acceptance Criteria

- [ ] The `github-actions` entry carries `cooldown` with `default-days: 3` and `exclude: ["neomjs/neo-agent-skills*"]`; nothing else in the file changes.

The falsifier is the run the config change triggers: #893 moves to `v1.x`. neomjs/neo#20619 carries the same check for the Engine.

## Out of Scope

The npm entry (#530), and the cron schedule (#891).

## Related

neomjs/neo#20618 · neomjs/neo#20619 · #530 · #891 · #893

Live latest-open + closed sweep: I read the latest 30 open issues (no Dependabot or cooldown hit) and ran a closed search ("dependabot cooldown") at 2026-11-08 22:2xZ. The hits were #891 and #530, neither equivalent; #530 is the npm half this completes. Backlog band: 44 open, orange, and no owning ticket exists.
A2A in-flight claim sweep: Grace offered me these at 22:26Z (67a83357), and I took them at 22:27Z (7506fa32).

Origin Session ID: dc9981ce-06c3-426d-ac3e-63aeb7f388fa

## Timeline

- 2026-11-08T22:27:44Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-08T22:27:46Z @neo-opus-vega added the `enhancement` label
- 2026-11-08T22:27:46Z @neo-opus-vega added the `ai` label
- 2026-11-08T22:27:46Z @neo-opus-vega added the `build` label
- 2026-11-08T22:28:59Z @neo-opus-vega cross-referenced by PR #897
### @jay23606 - 2026-11-09T12:00:06Z

Could I be assigned to this? The change is a small edit to the github-actions entry in .github/dependabot.yml, so I'd like to take it.

- 2026-11-09T12:10:06Z @tobiu referenced in commit `a12b3b0` - "ci(deps): the Skills workflow refs skip Dependabot's three-day hold (#896) (#897)

The github-actions entry had no cooldown, so Dependabot's default
three-day hold kept the neomjs/neo-agent-skills workflow refs at
v0.1.90 while the npm entry, exempted earlier, took the newest
release. The entry now carries the same exemption, as in the Engine's
neomjs/neo#20619."
- 2026-11-09T12:10:06Z @tobiu closed this issue

