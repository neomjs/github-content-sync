---
id: 891
title: 'Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-11-08T20:47:24Z'
updatedAt: '2026-11-08T21:51:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/891'
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
closedAt: '2026-11-08T21:51:55Z'
---
# Dependabot skips weekends, so releases shipped on Saturday or Sunday wait until Monday

## Context

`.github/dependabot.yml` schedules both ecosystems (npm, github-actions) with `interval: daily`, which Dependabot runs Monday to Friday only. The Institution's scheduled runs fire at 13:33Z. Both ecosystems ran on Oct 30 and on Nov 2–6, and neither ran on Oct 31–Nov 1 or Nov 7–8. The two Sunday runs in that window, on Nov 1 at 02:00Z and 18:46Z, were security updates for `dompurify`, `sharp` and `minimatch`, which run outside the schedule.

`neo-agent-skills` ships on weekends too: 1.0.0 and 1.1.0 came out on Sun Nov 8, and no Institution run proposes them before Monday. The Engine made the same change in neomjs/neo#20606, and Dependabot opened three update PRs within two minutes of that merge. The operator asked whether the Brain and the FM need it too (2026-11-08).

## The Problem

A weekday-only schedule can leave up to three days of skills releases, and of the shared-workflow tags the Institution calls, unproposed. The gap is widest when the team works through a weekend.

## The Architectural Reality

- Design authority: n/a — this changes a schedule and calls no behaviour wrong.
- Dependabot supports a cron schedule: `schedule.interval: cron` with `schedule.cronjob: "<expression>"`.
- The file's other policies stay as they are: majors are included, the 3-day cooldown excludes `neo-agent-skills`, the grouping keeps its `monaco-editor` and `neo-agent-skills` exclusions, and the coverage note on the `github:` refs stands.

## The Fix

Both ecosystems switch from `interval: daily` to `interval: cron` with `cronjob: "30 13 * * *"`: every day, at the hour the scheduled job already runs, so the organization's Dependabot PRs stay staggered on the 20 shared runners. A one-line header note says why `daily` was not enough.

Decision Record impact: none.

## Acceptance Criteria

- [ ] Both `schedule` blocks use `interval: cron` and `cronjob: "30 13 * * *"`; apart from the header note, nothing else in the file changes.

The weekend run itself is Dependabot's documented `cron` behavior, verified once for the organization under the Engine twin's Post-Merge Validation (neomjs/neo#20607).

## Out of Scope

Grouping, cooldown and the majors policy. `neo-agent-skills` keeps `daily`, for the reason given on neomjs/neo-agent-brain#1519.

## Related

neomjs/neo#20606 (the Engine's change) · neomjs/neo-agent-brain#1519 (the Brain's).

Live latest-open + closed sweep: I read the latest 20 open issues and ran a closed search ("dependabot schedule") at 2026-11-08 20:3xZ. Hits: #530 (the cooldown exclusion), #855 and #646; none is equivalent, and #530's Out of Scope keeps the schedule unchanged without ruling on it. An open search for "dependabot" finds no owning ticket or epic. Backlog band: 44 open, orange. No owning ticket exists, so this is filed.
A2A in-flight claim sweep: I read the latest 30 messages at 20:4xZ; no claim on Dependabot scheduling.
MC sweep: "Dependabot weekday-only schedule, weekend skills releases wait until Monday" returns only the Engine change's own session.
Own-assignment sweep: none of my open Institution issues touches `.github/dependabot.yml`.

Origin Session ID: dc9981ce-06c3-426d-ac3e-63aeb7f388fa

## Timeline

- 2026-11-08T20:47:25Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-08T20:47:26Z @neo-opus-vega added the `enhancement` label
- 2026-11-08T20:47:26Z @neo-opus-vega added the `ai` label
- 2026-11-08T20:47:27Z @neo-opus-vega added the `build` label
- 2026-11-08T20:47:50Z @neo-opus-vega cross-referenced by #1519
- 2026-11-08T20:52:07Z @neo-opus-vega cross-referenced by PR #892
- 2026-11-08T21:51:55Z @tobiu referenced in commit `39fda25` - "ci(deps): Dependabot runs every day, weekends included (#891) (#892)

`interval: daily` runs Monday to Friday, so a release that ships on a
weekend waited until Monday: skills 1.0.0 and 1.1.0 came out on Sunday.
Both ecosystems now run on a cron at 13:30 UTC every day, the hour the
scheduled job already used, so the organization's Dependabot runs stay
staggered. Majors, cooldown and grouping are unchanged."
- 2026-11-08T21:51:55Z @tobiu closed this issue
- 2026-11-08T22:27:45Z @neo-opus-vega cross-referenced by #896

