---
id: 123
title: 'Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-opus-vega
createdAt: '2026-11-08T20:47:33Z'
updatedAt: '2026-11-08T21:53:03Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/123'
author: neo-opus-vega
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
closedAt: '2026-11-08T21:53:03Z'
---
# Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday

## Context

`.github/dependabot.yml` schedules both ecosystems (npm, github-actions) with `interval: daily`, which Dependabot runs Monday to Friday only. DevIndex's scheduled runs fire at 02:05Z on weekdays, Nov 3–6 most recently.

`neo-agent-skills` ships on weekends too: 1.0.0 and 1.1.0 came out on Sun Nov 8, and no DevIndex run proposes them before Monday. The Engine made the same change in neomjs/neo#20606, and Dependabot opened three update PRs within two minutes of that merge. The operator asked which other organization repositories need it (2026-11-08).

## The Problem

A weekday-only schedule can leave up to three days of skills releases, and of `neo.mjs` releases, unproposed. The gap is widest when the team works through a weekend.

## The Architectural Reality

- Design authority: n/a — this changes a schedule and calls no behaviour wrong.
- Dependabot supports a cron schedule: `schedule.interval: cron` with `schedule.cronjob: "<expression>"`.
- The file's other policies stay as they are: majors are included, the 3-day cooldown excludes `neo-agent-skills`, and the grouping keeps its `monaco-editor` and `neo-agent-skills` exclusions.

## The Fix

Both ecosystems switch from `interval: daily` to `interval: cron` with `cronjob: "0 2 * * *"`: every day, at the hour the scheduled job already runs, so the organization's Dependabot PRs stay staggered on the 20 shared runners. A one-line header note says why `daily` was not enough.

Decision Record impact: none.

## Acceptance Criteria

- [ ] Both `schedule` blocks use `interval: cron` and `cronjob: "0 2 * * *"`; apart from the header note, nothing else in the file changes.

The weekend run itself is Dependabot's documented `cron` behavior, verified once for the organization under the Engine twin's Post-Merge Validation (neomjs/neo#20607).

## Out of Scope

Grouping, cooldown and the majors policy.

## Related

neomjs/neo#20606 (the Engine's change) · neomjs/neo-agent-brain#1519 (the Brain's).

Live latest-open + closed sweep: I read the open issues (2) and ran a closed search ("dependabot schedule") at 2026-11-08 20:3xZ. The one hit, #51, is the cooldown exclusion and isn't equivalent. Backlog band: 2 open, green.
A2A in-flight claim sweep: I read the latest 30 messages at 20:4xZ; no claim on Dependabot scheduling.
MC sweep: "Dependabot weekday-only schedule, weekend skills releases wait until Monday" returns only the Engine change's own session.
Own-assignment sweep: I have no open DevIndex issues.

Origin Session ID: dc9981ce-06c3-426d-ac3e-63aeb7f388fa

## Timeline

- 2026-11-08T20:47:33Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-08T20:47:34Z @neo-opus-vega added the `enhancement` label
- 2026-11-08T20:47:34Z @neo-opus-vega added the `ai` label
- 2026-11-08T20:47:35Z @neo-opus-vega added the `dependencies` label
- 2026-11-08T20:47:50Z @neo-opus-vega cross-referenced by #1519
- 2026-11-08T20:52:09Z @neo-opus-vega cross-referenced by PR #124
- 2026-11-08T21:53:03Z @tobiu referenced in commit `c9ceeaf` - "ci(deps): Dependabot runs every day, weekends included (#123) (#124)

`interval: daily` runs Monday to Friday, so a release that ships on a
weekend waited until Monday: skills 1.0.0 and 1.1.0 came out on Sunday.
Both ecosystems now run on a cron at 02:00 UTC every day, the hour the
scheduled job already used, so the organization's Dependabot runs stay
staggered. Majors, cooldown and grouping are unchanged."
- 2026-11-08T21:53:03Z @tobiu closed this issue
- 2026-11-08T22:27:52Z @neo-opus-vega cross-referenced by #128

