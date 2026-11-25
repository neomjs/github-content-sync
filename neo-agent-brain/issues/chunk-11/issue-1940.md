---
id: 1940
title: Name mc-server's remaining slow calls from the probe's first day
state: OPEN
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T19:09:12Z'
updatedAt: '2026-11-25T19:09:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1940'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1843
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
# Name mc-server's remaining slow calls from the probe's first day

## Context

#1843 names three hot read paths: `who_is_online`, `list_messages` and `query_recent_turns`. Its subs fixed the two that were measured, the wake-records listing (#1931) and the `directOnly` broadcast walk (#1935). #1933 added the instrument for the rest: `argsShape` on every start row, and `eventLoopBlocks` naming each block's calls and GC time. Per #1933's Out of Scope, each remaining hot query gets its own sub only once the probe names it.

## The Problem

#1843 still lists suspects nobody has attributed since the instrument shipped:
- `WakeSubscriptionService#_readReviewLifecycleLoad`'s 30-day `MESSAGE:` scan behind `who_is_online`;
- wide `box:'all'` / `status:'all'` mailbox reads;
- `query_recent_turns`' 51.8 s tail.

Filing fixes for them before measuring would be fixing guesses.

## The Architectural Reality

- Design authority: #1933's Out of Scope, "Each gets its own sub of #1843 once this probe names it."
- `get_memory_core_tool_metrics` serves `recentSlowCalls` and `unfinishedCalls` with `argsShape`, and `eventLoopBlocks` with `callIds` that join to those rows.

## The Fix

After the plane runs #1937 for a day, read `eventLoopBlocks` and `recentSlowCalls`. Group them by tool and argument shape, and post the attribution here. Then file one sub of #1843 per named shape, or close this with a receipt that no block named one.

## Acceptance Criteria

- [ ] AC-1: an attribution receipt covering at least 24 h of probe data. It counts blocks and slow calls per tool and shape, with GC time per block.
- [ ] AC-2: every shape responsible for a block of 10 s or more has its own sub of #1843, linked here, or the receipt shows none occurred.

## Out of Scope

Fixing any query here. CPU and heap sizing (#1929) consumes this profile.

## Related

Parent: #1843 · Instrument: #1933 / PR #1937 · Sibling: the post-cut verification sub · #1929

Live latest-open + closed sweep: as for its sibling, at 2026-11-25T19:09Z. No equivalent found.

Origin Session ID: 86f1f927-fcc8-4839-adab-bb17cfb5c265

## Timeline

- 2026-11-25T19:09:13Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T19:09:14Z @neo-opus-ada added the `bug` label
- 2026-11-25T19:09:14Z @neo-opus-ada added the `ai` label
- 2026-11-25T19:09:15Z @neo-opus-ada added the `performance` label
- 2026-11-25T19:09:15Z @neo-opus-ada added the `agent-os` label
- 2026-11-25T19:09:23Z @neo-opus-ada added parent issue #1843
- 2026-11-25T19:41:01Z @neo-gemini-julia cross-referenced by #1939

