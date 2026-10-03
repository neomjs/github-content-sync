---
id: 144
title: Each neo-agent-skills release reaches every consumer as a standalone Dependabot PR on its next run
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-10-03T22:20:04Z'
updatedAt: '2026-10-03T22:20:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/144'
author: neo-opus-vega
commentsCount: 0
parentIssue: 14
subIssues:
  - '[ ] 19392 Dependabot proposes each neo-agent-skills release on its next run, not three days later'
  - '[ ] 833 Dependabot proposes each neo-agent-skills release on its next run, not three days later'
  - '[ ] 530 Dependabot proposes each neo-agent-skills release on its next run, not three days later'
  - '[ ] 51 Dependabot proposes each neo-agent-skills release on its next run, not three days later'
subIssuesCompleted: 0
subIssuesTotal: 4
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Each neo-agent-skills release reaches every consumer as a standalone Dependabot PR on its next run

## Context

D#19384's correction published as 0.1.29 on 2026-10-03 at 19:17Z and was loaded in no consumer. Engine sat on 0.1.19, Brain on 0.1.23, Institution on 0.1.24 and devindex on `^0.1.18`. #140 needed three hand pins (neomjs/neo#19391, neomjs/neo-agent-brain#831, neomjs/neo-agent-institution#526) and still has none for devindex. Planner disposition: Emmy accepted this outcome on 2026-10-03 as a delivery-durability follow-up, not a new gate on #140.

## The Problem

All four consumers run Dependabot's npm ecosystem daily with `neo-agent-skills` ungrouped, and none configures a cooldown. Since 2026-07-14, Dependabot applies a default three-day cooldown to version updates when none is configured ([changelog](https://github.blog/changelog/2026-07-14-dependabot-version-updates-introduce-default-package-cooldown/); [options reference](https://docs.github.com/en/code-security/reference/supply-chain-security/dependabot-options-reference#cooldown): "If not specified, Dependabot applies a default cooldown of 3 days").

The job logs show it on this package:
- Brain run `36965072944` and Engine run `36979044828` (2026-10-02) both log "Filtered out 4 versions due to cooldown", and the Engine run adds "No update needed for neo-agent-skills 0.1.19".
- Engine run `36392141231` (2026-09-28) skipped 0.1.19 at 2.9 days old.

A green job and an empty PR queue therefore read as current while every consumer lags at least three days, and a same-day correction needs a hand pin in each repository. The operator has ruled against that for org-internal consumers: relayed by Vega, 2026-09-21, after merging neomjs/github-content-sync#17, org-internal consumers run the latest, with "manual updates that no-one will ever do again". On 2026-09-24 he said Dependabot "does pick up new skills package versions" (recorded by Grace).

## The Architectural Reality

- Declared ranges: neo `0.1.19` (exact), Brain `^0.1.23`, Institution `^0.1.24`, devindex `^0.1.18`. Each `.github/dependabot.yml` has one npm entry with `interval: daily` and `neo-agent-skills` in the group's `exclude-patterns`.
- `cooldown` governs version updates only; security updates are never delayed by it.
- Merges stay human-gated, so the supply-chain guard on our own package is the review and the merge, not the cooldown.

## The Fix

In each consumer's existing npm entry, exempt only `neo-agent-skills`:

```yaml
    cooldown:
      default-days: 3
      exclude: ["neo-agent-skills"]
```

External packages keep their three days, and the schedule, grouping and target branch are unchanged. There is one leaf and PR per consumer: neo, Brain, Institution and devindex.

## Acceptance Criteria

- [ ] AC-1 Each consumer's leaf merges with only that npm entry changed.
- [ ] AC-2 A real later release, younger than three days at the time, reaches every consumer's next scheduled Dependabot run as eligible and is proposed in a standalone new or updated PR at the latest version. Each consumer needs the job log line plus the PR and its version. A no-filter log alone, or a run with no newer release, does not discharge this; queue or API blockers are recorded as they are.
- [ ] AC-3 Read from each merged config: external packages keep the cooldown, and CI, review and the human merge are unchanged.

## Out of Scope

- #140's manual pin, load and replay close.
- The reusable workflow coordinate (#80).
- A cross-repository push from Skills (rejected in #80).
- Recipient loading: a PR is delivery, not adoption.

## Related

Parent #14 · #140 · #80 · neomjs/neo#19391 · neomjs/neo-agent-brain#831 · neomjs/neo-agent-institution#526 · D#19384

Sweeps: the latest 20 open Skills issues at 2026-10-03T22:19Z show no equivalent; #140 (integration close) and #80 (workflow coordinate) are named above. A2A: the planner's acceptance, and no competing claim. Memory Core: the planner's disposition draft and the two operator statements above. Own assignments: none overlapping. Structure map: N/A (consumer configuration only).

Decision Record impact: none.

Origin Session ID: 0ef9cb1f-7610-4bfa-a498-43f8a9ba640c
Retrieval Hint: "Dependabot default cooldown three days neo-agent-skills consumers lag exclude next run delivery"

## Timeline

- 2026-10-03T22:20:06Z @neo-opus-vega added the `enhancement` label
- 2026-10-03T22:20:06Z @neo-opus-vega added the `ai` label
- 2026-10-03T22:20:16Z @neo-opus-vega added parent issue #14
- 2026-10-03T22:22:04Z @neo-opus-vega cross-referenced by #19392
- 2026-10-03T22:22:10Z @neo-opus-vega cross-referenced by #833
- 2026-10-03T22:22:17Z @neo-opus-vega cross-referenced by #530
- 2026-10-03T22:22:22Z @neo-opus-vega cross-referenced by #51
- 2026-10-03T22:22:52Z @neo-opus-vega added sub-issue #19392
- 2026-10-03T22:22:53Z @neo-opus-vega added sub-issue #833
- 2026-10-03T22:22:54Z @neo-opus-vega added sub-issue #530
- 2026-10-03T22:22:55Z @neo-opus-vega added sub-issue #51
- 2026-10-03T22:26:42Z @neo-opus-vega cross-referenced by PR #19393
- 2026-10-03T22:26:44Z @neo-opus-vega cross-referenced by PR #834
- 2026-10-03T22:26:46Z @neo-opus-vega cross-referenced by PR #531
- 2026-10-03T22:26:47Z @neo-opus-vega cross-referenced by PR #52

