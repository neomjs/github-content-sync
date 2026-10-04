---
id: 836
title: A wake receiver step that never settles is named stuck and the receiver restarts
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T11:36:36Z'
updatedAt: '2026-10-04T15:03:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/836'
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
closedAt: '2026-10-04T15:03:28Z'
---
# A wake receiver step that never settles is named stuck and the receiver restarts

Sub of #503 (its AC-8, from Grace's diagnosis [5979337947](https://github.com/neomjs/neo-agent-brain/issues/503#issuecomment-5979337947); intake [5979413659](https://github.com/neomjs/neo-agent-brain/issues/503#issuecomment-5979413659)). Split out so a PR can close it without closing #503's other ACs; the same shape as #734 for #503 AC-4.

## Context

From 2026-10-03 23:09Z the host wake receiver's `state.accept` path and its manifest sweep never settled. For twelve hours it answered nothing.
- **Plane log** (`mc-server-2026-10-04.log`): every delivery from 09:49Z ends `aborted due to timeout`. The plane marked every seat's route `degraded` between 10:03:29Z and 10:31:53Z.
- **Receiver:** no record and no reload line after 23:09Z.
- **Isolation probes, 11:0xZ:** an unsigned POST got `401` in 1 ms, and a validly signed POST with a mismatched envelope got `409` in 76 ms. Known-route requests therefore stop at or after `state.accept`.
- **Observed recovery:** a `launchctl kickstart -k` at 11:13Z (Grace). Deliveries succeeded from 11:15:57Z.

**Unknown:** which step hung. A `sample` of the live process at about 11:05Z showed its four libuv workers idle, so it was not a blocked filesystem syscall at that instant.

## The Problem

A step that never settles throws nothing and logs nothing. The receiver keeps answering other paths, so every probe reads it alive while it accepts nothing. Today's cost: every seat's route was withdrawn, the team idled, and the operator woke eight seats by hand.

## The Architectural Reality

- `ai/daemons/wake/receiver.mjs` holds the request handler (`state.accept` before the `202`), the serial drain (`state.list`, `state.transition`, the context probe, `dispatch`), and the manifest reload chain (`serialize`). None of them is bounded.
- The supervisor already exists. `com.neomjs.agent-os-wake` runs the receiver with `KeepAlive: true` and `ThrottleInterval: 10`, so an exited receiver is back in about 10 s.
- **The restart must stay launchd-owned.** The osascript adapter's Accessibility grant follows the process that launched the receiver. An agent-spawned replacement accepts wakes and silently fails every GUI hop (incident of 2026-08-01). The `:3199` receiver is operator-owned host infrastructure: it may restart itself through its own LaunchAgent, and an agent never replaces it.

## The Fix

A stuck-step watchdog inside `createWakeReceiver`.
- `track(step, promise, boundMs?)` holds each accept, reload, drain step and dispatch in a set while it is in flight.
- An interval names the first step that outlives its bound, logs it, and calls `onStuck` once. The bounds (`STEP_BOUNDS_MS`): accept 15 s, reload 15 s, drain 60 s. A dispatch gets its **route's own `attemptTimeoutMs` plus 30 s of headroom**, since the adapter already races every attempt against that budget and the manifest admits up to 300 s.
- `startWakeReceiver`'s production `onStuck` exits non-zero, and launchd restarts the receiver fresh. Library callers and tests inject their own `onStuck`.

The bounds are well above normal costs: a full scan of the live 10,046 records takes about 550 ms.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Evidence |
| --- | --- | --- | --- | --- |
| `createWakeReceiver({onStuck, stepBoundsMs, watchdogIntervalMs})` | `ai/daemons/wake/receiver.mjs` | `onStuck({step, subscriptionId, ageMs})` fires once, for the first in-flight step past its bound. `subscriptionId` names a stuck dispatch's route. The default `null` reports by log line only and never exits. | Without a callback, a `STUCK:` error line is the whole report | spec: never-settling accept and dispatch arms |
| returned `track(step, promise, boundMs?)` | same | Holds a promise under the watchdog until it settles, with the same outcome. `boundMs` defaults to `stepBoundsMs[step]` | — | `startWakeReceiver`'s reload chain |
| `STEP_BOUNDS_MS`, `STEP_WATCHDOG_INTERVAL_MS` | same | Accept 15 s, reload 15 s, drain 60 s. Dispatch = the route's `attemptTimeoutMs` (manifest-admitted ≤ 300 s) + 30 s headroom. Polled every 5 s | — | boundary arms: 1000 ms on a 1500 ms route stays silent; a never-settling dispatch trips only after 1500 ms + headroom |
| `startWakeReceiver({onStuck})` | same | Default `process.exit(1)`. The LaunchAgent (`KeepAlive`, `ThrottleInterval` 10) restarts it; nothing in the receiver spawns a replacement | An injected `onStuck` replaces the exit | code |
| Disposal | same | The interval is `unref`'d and cleared when the `server` closes | — | spec `afterEach` closes every server |
| A dispatch interrupted by the exit | `WakeReceiverState.recoverInterrupted` | A record left `dispatching` is terminalized `unknown` at the next boot, never replayed, so no GUI side effect repeats | — | existing behavior, unchanged |
| Host rollout | #503 | Post-merge: pull the host receiver checkout, `launchctl kickstart -k` with the operator's agreement, and record one delivered wake | — | L3, owned by #503 |

**Recovery-time boundary.** Detection takes the step's bound plus at most one poll: accept or reload ≤ 20 s, drain ≤ 65 s, dispatch ≤ the route's budget + 35 s. Then launchd restarts after its throttle, plus node's startup. This leaf promises no request count and no re-arming. A route the plane degraded meanwhile stays degraded until its owner resumes it, and #837 makes that visible.

## Acceptance Criteria

- [ ] An accept that never settles is named stuck, as `accept`, within its bound. Red-first against the unchanged receiver.
- [ ] A dispatch that never settles is named stuck, as `dispatch`, after its wake was accepted with `202`. Red-first.
- [ ] A dispatch is never stuck inside its route's own `attemptTimeoutMs`, however far that exceeds the headroom. Past the budget plus the headroom it is, and not before. Red-first against a fixed dispatch bound (round 1, Euclid's RA-1).
- [ ] Non-vacuity: a delivered wake with every step settling trips nothing.
- [ ] `startWakeReceiver` exits non-zero on a stuck step by default, and the restart is the LaunchAgent's. No agent-run replacement process.
- [ ] Post-merge, host: the receiver checkout under `/Users/Shared/agent-os` is pulled to the merge and the receiver restarted through `launchctl kickstart -k` with the operator's agreement. One delivered wake after the restart is the receipt.

## Out of Scope

- **Startup runs before the watchdog exists** (Grace's design read). A hang in the manifest load, `state.init()` or `recoverInterrupted()` leaves a process that never listens and that launchd keeps alive. The plane sees connection refusals, not timeouts, so it is visible. It is unbounded here.
- Surfacing on `who_is_online` / `healthcheck`: the receiver's own liveness, including this watchdog's restart count, is #841. A withdrawn route, or one missing from the receiver's manifest, is #837.
- Naming the step that hung today. The watchdog's log line will name it the next time.
- Brain #30 (stop driving the GUI): the delivery-side root fix.
- Record retention (10,046 never-pruned records, about 550 ms per drain pass).

## Avoided Traps

- **A per-request 503 instead of an exit.** It answers the one request but leaves the wedged process serving, and the next accept hangs too.
- **An agent restarting the host receiver** (the Accessibility trap above).

Decision Record impact: aligned-with ADR 0025/0026. The receiver detects its own stuck state, and the restart actuator is the OS supervisor that already owns the process.

Structure map: no new file; owning module `ai/daemons/wake/receiver.mjs`.

Sweeps: live latest-20 open Brain issues read 11:3xZ, none equivalent (adjacent: #768, #609); A2A last 60 min: my own lane claim 11:22Z and Grace's reader yes; Memory Core: the 2026-08-01 Accessibility incident (cited above); own assignments: #503 (parent).

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "wake receiver stuck step watchdog launchd restart accept never settles"





## Timeline

- 2026-10-04T11:36:37Z @neo-opus-vega added the `bug` label
- 2026-10-04T11:36:37Z @neo-opus-vega added the `ai` label
- 2026-10-04T11:36:38Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T11:36:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T11:37:31Z @neo-opus-vega cross-referenced by #503
- 2026-10-04T11:39:39Z @neo-opus-vega cross-referenced by PR #838
- 2026-10-04T11:39:59Z @neo-opus-vega cross-referenced by #30
- 2026-10-04T11:48:19Z @neo-opus-ada cross-referenced by #15000
- 2026-10-04T12:29:44Z @neo-opus-vega referenced in commit `a614e8e` - "fix(wake): a dispatch is stuck only past its own route's attempt budget plus headroom, never inside it (#836)

Round 1 (Euclid): the manifest admits attemptTimeoutMs up to 300 s, so a
fixed 3-minute dispatch bound could call a still-permitted attempt stuck and
restart the shared receiver. The adapter already races every attempt against
its route's attemptTimeoutMs, so the watchdog now bounds a dispatch by that
budget plus 30 s of settlement headroom, carried per tracked entry.

A 1000 ms dispatch on a 1500 ms route no longer trips. A never-settling one
trips only after 1500 ms plus the headroom."
- 2026-10-04T12:31:16Z @neo-opus-vega referenced in commit `e636822` - "fix(wake): a stuck dispatch names the route it serves (#836)

Grace's design read: the STUCK line and the onStuck payload carry the
subscriptionId of a stuck dispatch, so the next incident names the
route whose adapter hung."
- 2026-10-04T12:31:58Z @neo-opus-vega cross-referenced by #837
- 2026-10-04T13:32:29Z @neo-opus-ada cross-referenced by #148
- 2026-10-04T13:50:19Z @neo-opus-ada cross-referenced by #147
- 2026-10-04T13:59:43Z @neo-opus-vega cross-referenced by #841
- 2026-10-04T14:44:29Z @neo-opus-grace cross-referenced by PR #845
- 2026-10-04T15:03:28Z @tobiu referenced in commit `de222e4` - "fix(wake): a receiver step that never settles is named stuck, and the receiver restarts (#836) (#838)

* fix(wake): a receiver step that never settles is named stuck and the receiver exits for its supervisor to restart it (#836)

From 2026-10-03 23:09Z the host receiver's accept path and its manifest
sweep never settled. It answered nothing for twelve hours, every delivery
timed out, the plane degraded every seat's route, and no surface said so.
A restart cured it. Which step hung is unknown.

So the receiver bounds its steps instead of guessing. Accept, the
manifest reload, each drain step and each dispatch are held under a
watchdog, and the first one that outlives its bound is logged by name and
handed to onStuck once. In production that exits non-zero, and the
LaunchAgent (KeepAlive, 10 s throttle) brings back a fresh process. The
same fault now costs one request and about ten seconds.

Red-first: an accept and a dispatch that never settle each name
themselves stuck within their bound. A delivered wake trips nothing.

* fix(wake): a dispatch is stuck only past its own route's attempt budget plus headroom, never inside it (#836)

Round 1 (Euclid): the manifest admits attemptTimeoutMs up to 300 s, so a
fixed 3-minute dispatch bound could call a still-permitted attempt stuck and
restart the shared receiver. The adapter already races every attempt against
its route's attemptTimeoutMs, so the watchdog now bounds a dispatch by that
budget plus 30 s of settlement headroom, carried per tracked entry.

A 1000 ms dispatch on a 1500 ms route no longer trips. A never-settling one
trips only after 1500 ms plus the headroom.

* fix(wake): a stuck dispatch names the route it serves (#836)

Grace's design read: the STUCK line and the onStuck payload carry the
subscriptionId of a stuck dispatch, so the next incident names the
route whose adapter hung."
- 2026-10-04T15:03:28Z @tobiu closed this issue

