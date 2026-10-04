---
id: 530
title: 'Dependabot proposes each neo-agent-skills release on its next run, not three days later'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T22:22:15Z'
updatedAt: '2026-10-04T16:53:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/530'
author: neo-opus-vega
commentsCount: 0
parentIssue: 144
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-04T16:53:12Z'
---
# Dependabot proposes each neo-agent-skills release on its next run, not three days later

## Context

This is the Institution's leaf of neomjs/neo-agent-skills#144: each Skills release reaches every consumer as a standalone Dependabot PR on its next run. The planner (Emmy) accepted it on 2026-10-03.

## The Problem

`.github/dependabot.yml`'s npm entry runs daily with `neo-agent-skills` excluded from the group and configures no cooldown. Dependabot's default three-day cooldown for version updates (since 2026-07-14) therefore filters every new release for three days; the Engine's and Brain's 2026-10-02 runs log it on this package. The Institution's last Dependabot Skills bump was 0.1.12 → 0.1.14 on 2026-09-24, and the move to 0.1.24 and now 0.1.29 (#526) was by hand.

## The Fix

Add to the npm entry:

```yaml
    cooldown:
      default-days: 3
      exclude: ["neo-agent-skills"]
```

Nothing else changes. External packages keep three days, and security updates were never delayed.

## Acceptance Criteria

- [ ] AC-1 The npm entry carries this cooldown. The schedule, groups, `exclude-patterns` and the `github-actions` ecosystem are unchanged (the diff).
- [ ] AC-2 (post-merge) This repository's delivery receipt lands on neomjs/neo-agent-skills#144 AC-2.

## Out of Scope

The `github-actions` ecosystem and the reusable-workflow coordinate (neomjs/neo-agent-skills#80).

Related: neomjs/neo-agent-skills#144 · #526

Decision Record impact: none. Sweeps: the latest 20 open Institution issues at 2026-10-03T22:22Z showed no equivalent, and A2A carried no claim. Structure map: N/A (repository configuration only).

Origin Session ID: 0ef9cb1f-7610-4bfa-a498-43f8a9ba640c

## Timeline

- 2026-10-03T22:22:17Z @neo-opus-vega added the `enhancement` label
- 2026-10-03T22:22:17Z @neo-opus-vega added the `ai` label
- 2026-10-03T22:22:17Z @neo-opus-vega added the `dependencies` label
- 2026-10-03T22:22:54Z @neo-opus-vega added parent issue #144
- 2026-10-03T22:23:20Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T22:26:46Z @neo-opus-vega cross-referenced by PR #531
- 2026-10-04T09:49:36Z @neo-opus-vega referenced in commit `996c91a` - "ci(dependabot): the cooldown rationale fits two lines (#530)"
- 2026-10-04T16:53:12Z @tobiu referenced in commit `e4240b7` - "ci(dependabot): each neo-agent-skills release is proposed on the next run, not three days later (#530) (#531)

* ci(dependabot): each neo-agent-skills release is proposed on the next run, not three days later (#530)

Dependabot holds every version update for three days when no cooldown is configured. Exempts only neo-agent-skills; every other package keeps the default, now explicit.

* ci(dependabot): the cooldown rationale fits two lines (#530)"
- 2026-10-04T16:53:12Z @tobiu closed this issue

