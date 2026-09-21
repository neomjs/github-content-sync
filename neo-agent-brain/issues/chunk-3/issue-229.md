---
id: 229
title: Delete the orphaned review-cost meter
state: CLOSED
labels:
  - ai
  - refactoring
  - testing
  - agent-os
  - tech-debt
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T04:17:38Z'
updatedAt: '2026-08-29T11:33:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/229'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 191
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T11:33:22Z'
---
# Delete the orphaned review-cost meter

Child of #191.

## Problem

`ai/scripts/diagnostics/review-cost-meter.mjs` is a 584-line local diagnostic with a 354-line private test suite. It has no npm command, workflow invocation, runbook, production caller, or retained artifact consumer. Outside its own test, the only reference is one catalog-style comment in another diagnostic.

Keeping 938 lines to preserve a command no supported surface can invoke is legacy retention, not product capability.

## Scope

Delete the meter and its dedicated test. Remove the stale catalog comment reference. Add no replacement, migration, census, ledger, or forwarding alias.

## Acceptance criteria

- [ ] The meter module and dedicated spec are absent.
- [ ] Tracked source contains zero `review-cost-meter` / `ReviewCostMeter` references.
- [ ] No package script, workflow, runbook, or production consumer is removed because none exists.
- [ ] Relevant lint and unit collection remain green.
- [ ] Production and test LOC decrease by 938 lines, excluding the one comment edit.

## Timeline

- 2026-08-29T04:17:39Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T04:17:40Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T04:17:40Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-29T04:17:40Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T04:17:40Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T04:17:40Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-29T04:21:07Z @neo-gpt-emmy cross-referenced by PR #230
- 2026-08-29T10:44:01Z @tobiu referenced in commit `a58ff96` - "chore(diagnostics): delete orphaned review-cost meter (#229)"
- 2026-08-29T11:03:06Z @tobiu referenced in commit `10f426b` - "chore(diagnostics): delete orphaned review-cost meter (#229)"
### @neo-gpt-emmy - 2026-08-29T11:33:21Z

Closing as wrong-premise. The claims that this meter has no runbook or supported consumer are false: neo-agent-skills/.agents/skills/pr-review/audits/review-cost-circuit-breaker.md documents the command, and neomjs/neo#15257 / PR #15307 make its telemetry part of the review-budget contract. Any follow-up must preserve the capability and re-home or simplify it rather than delete it without replacement.

- 2026-08-29T11:33:22Z @neo-gpt-emmy closed this issue
- 2026-08-29T11:37:10Z @neo-opus-vega cross-referenced by #233
- 2026-08-31T00:18:29Z @neo-gpt-emmy cross-referenced by #191
- 2026-09-06T18:31:08Z @neo-opus-grace cross-referenced by #70

