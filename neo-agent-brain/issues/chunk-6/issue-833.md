---
id: 833
title: 'Dependabot proposes each neo-agent-skills release on its next run, not three days later'
state: OPEN
labels:
  - enhancement
  - dependencies
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T22:22:09Z'
updatedAt: '2026-10-03T22:23:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/833'
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
---
# Dependabot proposes each neo-agent-skills release on its next run, not three days later

## Context

This is the Brain's leaf of neomjs/neo-agent-skills#144: each Skills release reaches every consumer as a standalone Dependabot PR on its next run. The planner (Emmy) accepted it on 2026-10-03.

## The Problem

`.github/dependabot.yml`'s npm entry runs daily with `neo-agent-skills` excluded from the group and configures no cooldown. Its own comment says why the package is ungrouped: "Which rules an agent reads is a reviewed fact or it is an accident." Dependabot's default three-day cooldown for version updates (since 2026-07-14) now holds every release back from that review. Run `36965072944` (2026-10-02, at 0.1.23) logged "Filtered out 4 versions due to cooldown".

## The Fix

Add to the npm entry:

```yaml
    cooldown:
      default-days: 3
      exclude: ["neo-agent-skills"]
```

Nothing else changes. External packages, including the exact-pinned Brain tier, keep three days, and security updates were never delayed.

## Acceptance Criteria

- [ ] AC-1 The npm entry carries this cooldown. The schedule, groups, `exclude-patterns` and the `github-actions` ecosystem are unchanged (the diff).
- [ ] AC-2 (post-merge) This repository's delivery receipt lands on neomjs/neo-agent-skills#144 AC-2.

## Out of Scope

The `github-actions` ecosystem and the reusable-workflow coordinate (neomjs/neo-agent-skills#80).

Related: neomjs/neo-agent-skills#144 · neomjs/neo-agent-skills#140

Decision Record impact: none. Sweeps: the latest 20 open Brain issues at 2026-10-03T22:22Z showed no equivalent, and A2A carried no claim. Structure map: N/A (repository configuration only).

Origin Session ID: 0ef9cb1f-7610-4bfa-a498-43f8a9ba640c

## Timeline

- 2026-10-03T22:22:11Z @neo-opus-vega added the `enhancement` label
- 2026-10-03T22:22:11Z @neo-opus-vega added the `dependencies` label
- 2026-10-03T22:22:11Z @neo-opus-vega added the `ai` label
- 2026-10-03T22:22:53Z @neo-opus-vega added parent issue #144
- 2026-10-03T22:23:18Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T22:26:44Z @neo-opus-vega cross-referenced by PR #834

