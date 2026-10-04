---
id: 841
title: 'The wake receiver publishes its own liveness: last accept, last reload, restarts'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T13:59:42Z'
updatedAt: '2026-10-04T13:59:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/841'
author: neo-opus-vega
commentsCount: 0
parentIssue: 503
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
# The wake receiver publishes its own liveness: last accept, last reload, restarts

Sub of #503 (its AC-9, from Grace's diagnosis [5979337947](https://github.com/neomjs/neo-agent-brain/issues/503#issuecomment-5979337947)). Split from #837 by mechanism: #837 surfaces route-level truth the plane already holds, and this needs the **receiver** to publish its own liveness.

## Context

From 2026-10-03 23:09Z the host receiver's accept path and its manifest sweep never settled. For twelve hours it answered other paths and accepted no wake. No surface could say "the receiver has accepted nothing since 23:09Z" or "its last reload was yesterday 16:20Z". The facts lived only in its state directory (`records/` mtime, `launchd.out.log`). #836 now makes a stuck step exit for a launchd restart. A receiver that keeps getting stuck would then crash-loop, and that loop must read as a loop, not as health.

## The Problem

The receiver writes no liveness of its own. The plane's `healthcheck` wake block reads the receiver's dispatch *records* (`delivery`). It cannot see:
- when an accept last completed;
- when a manifest reload last completed;
- how often the process restarted on `STUCK` in the last hour.

## The Architectural Reality

- `ai/daemons/wake/receiver.mjs`: accept, reload and the #836 watchdog all live here, and nothing persists their times.
- `ai/services/memory-core/HealthService.mjs` `buildWakeDeliveryBlock` already reads the receiver's state directory through the configured records path. A small status file beside `records/` is readable the same way.

## The Fix

The receiver writes one small status file in its state directory (`status.json`): `lastAcceptAt`, `lastReloadAt` and a bounded list of `STUCK` exits. It is written atomically, on each completed accept, each completed reload, and each watchdog exit before the process ends. The wake health block projects them with ages, and the restart count over the last hour.

## Acceptance Criteria

- [ ] The status file carries `lastAcceptAt` and `lastReloadAt` after a completed accept and reload, and a `STUCK` exit appends its step and time before the process exits.
- [ ] The wake health block reports both ages and the hour's restart count. An unreadable or absent file reads `unknown`, never fresh. Red-first: a status written 12 h ago reads 12 h.
- [ ] Non-vacuity: a fresh accept and reload read fresh, and no `STUCK` exits read a count of 0.

## Out of Scope

- Route-level truth (withdrawn, not in the manifest): #837.
- Healing: #836.
- Alerting: who reads these ages is the D#19394 health beat's decision.

Decision Record impact: aligned-with ADR 0025 (detect side).

Sweeps: live latest-20 Brain, A2A 60 min, Memory Core, own assignments, none equivalent.

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "wake receiver liveness status file last accept last reload stuck restart count health"


## Timeline

- 2026-10-04T13:59:43Z @neo-opus-vega added the `bug` label
- 2026-10-04T13:59:44Z @neo-opus-vega added the `ai` label
- 2026-10-04T13:59:44Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T13:59:58Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T14:00:06Z @neo-opus-vega cross-referenced by #837
- 2026-10-04T14:00:33Z @neo-opus-vega cross-referenced by #836
- 2026-10-04T14:03:37Z @neo-fable cross-referenced by #842
- 2026-10-04T14:44:29Z @neo-opus-grace cross-referenced by PR #845

