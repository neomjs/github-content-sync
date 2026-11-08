---
id: 1519
title: 'Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday'
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-11-08T20:47:07Z'
updatedAt: '2026-11-08T20:51:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1519'
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
---
# Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday

## Context

`.github/dependabot.yml` schedules both ecosystems (npm, github-actions) with `interval: daily`, which Dependabot runs Monday to Friday only. The Brain's scheduled runs fire at 04:33Z. Both ecosystems ran on Oct 30 and on Nov 2–6, and neither ran on Oct 31–Nov 1 or Nov 7–8. The one weekend run in that window, Sun Nov 1 at 22:59Z, was a security update for `autoprefixer` and `postcss`, which runs outside the schedule.

`neo-agent-skills` ships on weekends too: 1.0.0 and 1.1.0 came out on Sun Nov 8, and no Brain run proposes them before Monday. The Engine made the same change in neomjs/neo#20606, and Dependabot opened three update PRs within two minutes of that merge. The operator asked whether the Brain and the FM need it too (2026-11-08).

## The Problem

A weekday-only schedule can leave up to three days of skills releases, and of the shared-workflow tags the Brain calls, unproposed. The gap is widest when the team works through a weekend.

## The Architectural Reality

- Design authority: n/a — this changes a schedule and calls no behaviour wrong.
- Dependabot supports a cron schedule: `schedule.interval: cron` with `schedule.cronjob: "<expression>"`.
- The file's other policies stay as they are: majors are included, the 3-day cooldown excludes `neo-agent-skills`, the grouping keeps its four exclusions, and the coverage note on the `neo.mjs` tarball pin stands.

## The Fix

Both ecosystems switch from `interval: daily` to `interval: cron` with `cronjob: "30 4 * * *"`: every day, at the hour the scheduled job already runs. Each repository keeps its own hour (Engine 07:30, Brain 04:30, Institution 13:30, devindex 02:00), so the organization's Dependabot PRs keep reaching the 20 shared runners at different times. A one-line header note says why `daily` was not enough.

Decision Record impact: none.

## Acceptance Criteria

- [ ] Both `schedule` blocks use `interval: cron` and `cronjob: "30 4 * * *"`; apart from the header note, nothing else in the file changes.

The weekend run itself is Dependabot's documented `cron` behavior, verified once for the organization under the Engine twin's Post-Merge Validation (neomjs/neo#20607).

## Out of Scope

- Grouping, cooldown and the majors policy.
- `neo-agent-skills` keeps `daily`. Every maintainer PR there must carry a new version, and each merge publishes it (`scripts/check-version-bump.mjs`). So this change alone would ship a release with no package change and open a bump PR in every consumer. Skills consumes no Skills release through npm, so its weekend gap costs little.

## Related

neomjs/neo#20606 (the Engine's change) · neomjs/neo-agent-institution#891 · neomjs/devindex#123.

Live latest-open + closed sweep: I read the latest 20 open issues and ran a closed search ("dependabot schedule") at 2026-11-08 20:3xZ. Hits: #833 (the cooldown exclusion) and #888; neither is equivalent, and #833's Out of Scope keeps the schedule unchanged without ruling on it. An open search for "dependabot" finds no owning ticket or epic. Backlog band: 170 open, red. No owning ticket exists, so this is filed.
A2A in-flight claim sweep: I read the latest 30 messages at 20:4xZ; no claim on Dependabot scheduling.
MC sweep: "Dependabot weekday-only schedule, weekend skills releases wait until Monday" returns only the Engine change's own session.
Own-assignment sweep: none of my open Brain issues touches `.github/dependabot.yml`.

Origin Session ID: dc9981ce-06c3-426d-ac3e-63aeb7f388fa

## Timeline

- 2026-11-08T20:47:08Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-08T20:47:09Z @neo-opus-vega added the `enhancement` label
- 2026-11-08T20:47:09Z @neo-opus-vega added the `ai` label
- 2026-11-08T20:47:10Z @neo-opus-vega added the `build` label
- 2026-11-08T20:47:26Z @neo-opus-vega cross-referenced by #891
- 2026-11-08T20:47:34Z @neo-opus-vega cross-referenced by #123
- 2026-11-08T20:52:04Z @neo-opus-vega cross-referenced by PR #1520
- 2026-11-08T20:52:07Z @neo-opus-vega cross-referenced by PR #892
- 2026-11-08T20:52:09Z @neo-opus-vega cross-referenced by PR #124

