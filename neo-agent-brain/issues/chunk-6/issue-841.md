---
id: 841
title: 'The wake receiver publishes its own liveness: last accept, last reload, restarts'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T13:59:42Z'
updatedAt: '2026-10-04T18:29:43Z'
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
closedAt: '2026-10-04T18:29:43Z'
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
- `ai/services/memory-core/HealthService.mjs` `buildWakeDeliveryBlock` reads the receiver's records through `readWakeDelivery` and the configured records path. A plane mounts only `records/` (measured: `…/wake/state/records → /app/wake-receiver-records`), so the account must live there.

## The Fix

The receiver keeps one small liveness account in its records directory, `records/receiver.liveness`. The name isn't `.json`, so neither record reader takes it. The account holds:
- its starts, the last 20;
- `lastAcceptAt`, stamped by each completed accept;
- `lastSweepAt`, stamped by each completed pass of the manifest sweep, which runs every 30 s with or without traffic;
- the last 20 `STUCK` exits.

It is replaced atomically on each event. A stuck exit is written synchronously, before the process exits, and the next start carries it forward. `readWakeDelivery` projects the account as `receiver`, with both ages and the last hour's starts and stuck exits, so the healthcheck wake block carries it.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Evidence |
| --- | --- | --- | --- | --- |
| `records/receiver.liveness` | `ai/daemons/wake/receiverLiveness.mjs` (`normalizeReceiverLiveness`) | `{starts, lastAcceptAt, lastSweepAt, stuckExits: [{step, subscriptionId?, at}]}`. ISO stamps, at least one start, both lists ≤ 20. The receiver is the only writer; it replaces the file whole and atomically. | A torn, unparseable or wrong-shaped account reads `null` | unit: torn file, four wrong shapes |
| `createReceiverLiveness({recordsDir, now, logger, fsModule, readTimeoutMs})` → `start`, `accepted`, `swept`, `stuck` | same | The start is stamped at creation. `start()` reads the previous account within `readTimeoutMs` (2 s) and carries it forward; later writes queue behind it. `stuck()` writes synchronously. | A read that times out or is invalid starts fresh. A failed write is logged, never thrown | unit arms |
| Startup | `startWakeReceiver` | `liveness.start()` is not awaited, so the watchdog and listener never wait on observation | — | startup control: a never-settling read still serves (401) |
| Accept stamp | `createWakeReceiver({onAccepted})` | Fires after each completed accept (202/200); a refused request stamps nothing | — | receiver arm |
| Sweep stamp | the serialized reload chain | Each completed pass stamps `lastSweepAt` inside the watchdog-tracked pass: a write that hangs is a stuck pass, so the process exits and launchd restarts it | — | receiver arm |
| Stuck exit | `startWakeReceiver`'s `onStuck` wrapper | Recorded synchronously before the production exit (or an injected `onStuck`); the next start keeps it | — | unit + restart arm |
| `readWakeDelivery().receiver` | `ai/services/memory-core/wakeDeliveryReader.mjs` | `startedAt`, `lastAcceptAt`/`lastAcceptAgeMs`, `lastSweepAt`/`lastSweepAgeMs`, `startsLastHour`, `stuckExitsLastHour`, `lastStuckExit` → healthcheck `features.wake.delivery.receiver` | `{state: 'unknown'}` when unconfigured, unreadable, absent or invalid | reader arms |
| Record exclusion | `WakeReceiverState#list`, `readWakeDelivery` | Both read only `.json`, so the account is never a dispatch record | — | reader + receiver arms |
| `stopWatchingManifest()` | `startWakeReceiver` | Returns the reload chain, which settles after the in-flight pass and its stamp | — | spec teardown |
| Rollout | #503 | The host receiver (deployed with #838) and the plane's reader both run it; then `delivery.receiver` reads a sweep age under a minute | — | L3, Residual-Owner #503 |

## Acceptance Criteria

- [ ] The account carries a start, `lastAcceptAt` after a completed accept, and `lastSweepAt` after a completed sweep pass. A `STUCK` exit is on disk before the process exits, and the next start keeps it. The receiver's startup never waits on the account: a read that never settles still leaves it serving.
- [ ] The wake health block's `delivery.receiver` reports both ages and the last hour's starts and stuck exits. An unreadable, absent or wrong-shaped account reads `unknown`, never fresh, and a start over it begins fresh without throwing. Red-first: an account written 12 h ago reads 12 h.
- [ ] Non-vacuity: a fresh accept and sweep read fresh, and no `STUCK` exit reads a count of 0. The account is never counted as a dispatch record.

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
- 2026-10-04T13:59:44Z @neo-opus-vega added parent issue #503
- 2026-10-04T13:59:58Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T14:00:06Z @neo-opus-vega cross-referenced by #837
- 2026-10-04T14:00:33Z @neo-opus-vega cross-referenced by #836
- 2026-10-04T14:03:37Z @neo-fable cross-referenced by #842
- 2026-10-04T14:44:29Z @neo-opus-grace cross-referenced by PR #845
- 2026-10-04T16:06:27Z @neo-fable-clio cross-referenced by #30
- 2026-10-04T16:13:40Z @neo-opus-vega cross-referenced by #503
- 2026-10-04T16:48:12Z @neo-opus-vega cross-referenced by PR #851
- 2026-10-04T17:51:50Z @neo-opus-vega referenced in commit `83d2a38` - "fix(wake): the receiver's liveness account is validated, and its startup never waits on it (#841)

A parseable account of the wrong shape (starts:{} or stuckExits:{})
threw in the writer's start, and the reader reported it as observed.
normalizeReceiverLiveness now owns the shape: lists must be lists, there
must be at least one valid start, and invalid entries are dropped.
Anything else reads unknown, and a start over it begins fresh.

startWakeReceiver no longer awaits the liveness start, so a read that
never settles cannot keep the receiver from listening. The start is
stamped at creation, its read is bounded (2 s), and later writes queue
behind it, so the previous account is still read before it is replaced."
- 2026-10-04T18:29:43Z @tobiu referenced in commit `7f895e8` - "feat(wake): the wake receiver keeps its own liveness, and healthcheck reads whether it accepts anything (#841) (#851)

* feat(wake): the wake receiver keeps its own liveness, and healthcheck reads whether it accepts anything (#841)

Twice on 2026-10-04 the host receiver stopped accepting wakes while
answering other paths, and nothing could say so. The receiver now keeps
records/receiver.liveness (a non-.json name, so no record reader takes
it). It records its starts, the last completed accept, the last
completed pass of its 30 s manifest sweep, and the stuck exits of
#838's watchdog. A stuck exit is written before the process exits.

readWakeDelivery projects the account as receiver, with ages and the
last hour's starts and stuck exits, so healthcheck's wake delivery
block carries it. An absent account reads unknown, never fresh.

* fix(wake): the receiver's liveness account is validated, and its startup never waits on it (#841)

A parseable account of the wrong shape (starts:{} or stuckExits:{})
threw in the writer's start, and the reader reported it as observed.
normalizeReceiverLiveness now owns the shape: lists must be lists, there
must be at least one valid start, and invalid entries are dropped.
Anything else reads unknown, and a start over it begins fresh.

startWakeReceiver no longer awaits the liveness start, so a read that
never settles cannot keep the receiver from listening. The start is
stamped at creation, its read is bounded (2 s), and later writes queue
behind it, so the previous account is still read before it is replaced."
- 2026-10-04T18:29:43Z @tobiu closed this issue

