---
id: 1918
title: 'No daemon, shell arm or wake route resolves through a pre-move seat path'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-24T20:50:25Z'
updatedAt: '2026-11-24T20:50:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1918'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
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
# No daemon, shell arm or wake route resolves through a pre-move seat path

## Context

#571's 47 subs are closed, and the per-seat move receipts exist (2026-10-09). Its terminal predicate still names one clause without a receipt: "Afterwards no machine daemon, shell arm or wake route resolves through a pre-move path." Under the operator's rule (24 Nov: an epic whose subs are all resolved is closed or gets new subs), that clause becomes this sub.

## The Problem

No audit has recorded that the operator's host is free of the seats' pre-move locations. A launchd agent, a cron line, a shell profile, a git hook or a wake subscription that still names an old clone would keep working by accident until that path is deleted.

## The Architectural Reality

- Seats live at `~/.neo-ai/agents/<agent-id>/` (#571).
- Wake subscriptions live on the plane; launchd agents, cron, shell profiles and git hooks live on the host.
- Design authority: #571's terminal predicate (quoted above).

## The Fix

Run one read-only audit on the operator's host, and record a receipt on #571 naming each surface checked and what it resolves to. Any pre-move hit becomes its own one-PR issue, linked here.

## Acceptance Criteria

- [ ] The receipt on #571 covers launchd agents, the crontab, the shell profiles, the seat clones' git hooks, and every seat's wake subscriptions.
- [ ] Every path they resolve is under `~/.neo-ai/agents/` or the plane; any hit is filed and linked.

## Out of Scope

The Docker plane cutover (#84) and seat features.

## Related

#571 (parent), #84.

Live latest-open + closed sweep: checked the latest 20 open issues plus an open and closed search (`pre-move path seat daemon wake route`) at 2026-11-24T20:50Z; #84 covers the plane, not seat paths. 154 open (red band); filed as a sub of #571, its owning epic.
A2A in-flight claim sweep: checked messages since 20:00Z; no claim.
MC sweep: the 2026-10 migration diaries (Vega, Clio, Emmy) record the moves and no path audit.
Own-assignment sweep: #571 is mine.

ownership: self-assigned.

Origin Session ID: 36593ade-49d5-4bf2-92e6-6e86d12beb9b
Retrieval Hint: "pre-move seat path audit launchd cron wake subscriptions 571"

## Timeline

- 2026-11-24T20:50:25Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-24T20:50:27Z @neo-opus-ada added the `enhancement` label
- 2026-11-24T20:50:28Z @neo-opus-ada added the `ai` label
- 2026-11-24T20:50:35Z @neo-opus-ada added parent issue #571

