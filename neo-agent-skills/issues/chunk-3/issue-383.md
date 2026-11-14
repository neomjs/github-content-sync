---
id: 383
title: Add centered community and package badge block to README.md
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-14T16:14:23Z'
updatedAt: '2026-11-14T16:47:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/383'
author: neo-gemini-maryam
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
closedAt: '2026-11-14T16:47:20Z'
---
# Add centered community and package badge block to README.md

## Context

Design authority: "neomjs/neo additionally has badges for slack and discord. these should be inside the brain, institution, skills, create-app repos too." (`@tobiu` operator directive, 2026-11-14; canonical badge markup on `neomjs/neo` `dev` `README.md` lines 13–14).

Unlike `neomjs/neo`, `neomjs/neo-agent-brain`, `neomjs/neo-agent-institution`, and `neomjs/create-app`, `neomjs/neo-agent-skills/README.md` opens directly with `# neo-agent-skills` at line 1 and has no `<p align="center">` badge block for its published npm package (`neo-agent-skills`), `LICENSE`, or the Neo.mjs Slack and Discord community channels.

- **Ownership:** Self-assigned (`@neo-gemini-maryam`); `good first issue` and `help wanted` omitted because this ticket is directly claimed under the operator's four-repo completeness directive (and maintainer PRs in `neo-agent-skills` require a synchronized release bump in `package.json` + `package-lock.json`).
- **Sweeps (`§1a`):**
  - Live latest-open + closed sweep: checked latest 20 open (35 open total — orange band; checked `#363` and `#359`, neither governs header badges) + closed search (`README Slack Discord`, `README badge`) at `2026-11-14T16:11:04Z`; no equivalent or prior exclusion found.
  - A2A in-flight claim sweep: checked latest 30 mailbox messages across all read-states at `2026-11-14T16:11:22Z`; offered directly to `@neo-gemini-maryam` by `@neo-fable-clio`, zero competing claims.
  - MC sweep: `"README missing Slack Discord badges neotericjs invite link"`, 5 results, no prior exclusion found (surfaces `@neo-fable-clio` lane offer `b0a75add`).
  - Own-assignment sweep: 2 open (`#368` Windows CRLF tests, `#381` outbound-post `scheduledFor`), bodies checked and non-overlapping.

## The Problem

Readers arriving on `neomjs/neo-agent-skills` (on GitHub or npmjs.com, since `README.md` is shipped in `"files"`) have no header badge block linking to the npm package version, MIT license, or the Neo.mjs Slack and Discord channels.

## The Architectural Reality

- `neomjs/neo/README.md` lines 13–14 (`dev`) holds the canonical Slack (`zt-6c50ueeu-3E1~M4T9xkNnb~M_prEEOA`) and Discord (`656620537514164249`) badge lines.
- `neomjs/neo-agent-skills` has `./LICENSE` and publishes `neo-agent-skills` to npm, and has no root `CONTRIBUTING.md` (so no `./CONTRIBUTING.md` badge link should be added).
- Per `scripts/check-version-bump.mjs`, every maintainer PR targeting `dev` in `neomjs/neo-agent-skills` must bump `version` in both `package.json` and `package-lock.json` (`1.9.1` → `1.9.2`) so `publish.yml` ships the updated `README.md` to npm.
- Decision Record impact: `none` (README front-door metadata + patch release bump only).

## The Fix

1. Prepend a compact `<p align="center">` badge block at the top of `README.md` (above `# neo-agent-skills`, matching `neomjs/create-app/README.md`) carrying the npm `Version` badge (`neo-agent-skills`), `License: MIT` badge (`./LICENSE`), and the verbatim Slack and Discord badge lines from `neomjs/neo/README.md` lines 13–14 (`dev`).
2. Bump `version` from `1.9.1` to `1.9.2` in `package.json` and `package-lock.json`.

## Acceptance Criteria

- [ ] AC-1: `README.md` opens with a centered `<p align="center">` badge block containing npm `Version` (`neo-agent-skills`), `License: MIT` (`./LICENSE`), and the exact Slack and Discord badge lines copied verbatim from `neomjs/neo/README.md` lines 13–14 (`dev`).
- [ ] AC-2: `package.json` and `package-lock.json` bump `version` to `1.9.2`, and `node scripts/check-version-bump.mjs --base origin/dev`, `npm run lint`, and `npm test` exit `0`.

## Out of Scope

- Adding a `CONTRIBUTING.md` file or badge (tracked separately on `neomjs/neo#20889`).
- Modifying skill corpus files under `.agents/skills/` or `README.md` prose sections (`#359`, `#363`).

## Avoided Traps

- Copying a `./CONTRIBUTING.md` badge into a repository without `CONTRIBUTING.md` (which would introduce a 404 link on GitHub and npm).
- Opening a maintainer PR in `neo-agent-skills` without bumping `package.json` and `package-lock.json`, which fails `check-version-bump.mjs` in CI.

## Related

- Related: neomjs/neo#20682

Origin Session ID: 362922fb-79d1-46e6-86ff-3bb7956ebec7
Retrieval Hint: "README Slack Discord community badges brain institution skills create-app"

## Timeline

- 2026-11-14T16:14:25Z @neo-gemini-maryam added the `documentation` label
- 2026-11-14T16:14:25Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-14T16:14:25Z @neo-gemini-maryam added the `ai` label
- 2026-11-14T16:15:13Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-14T16:20:03Z @neo-gemini-maryam cross-referenced by PR #384
- 2026-11-14T16:47:20Z @tobiu referenced in commit `80fe755` - "docs(readme): add centered community and package badge block (#383) (#384)"
- 2026-11-14T16:47:22Z @tobiu closed this issue

