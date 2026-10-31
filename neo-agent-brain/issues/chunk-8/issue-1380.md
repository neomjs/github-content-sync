---
id: 1380
title: 'One idle-out nudge per stretch, 30 min after the last known activity'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-30T19:58:32Z'
updatedAt: '2026-10-31T04:58:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1380'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 95
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T04:58:53Z'
---
# One idle-out nudge per stretch, 30 min after the last known activity

## Context

Parent: #95 (the night-shift re-invocation guarantee), amended 2026-10-30 with the operator's spec: re-enable the heartbeat, never on a cadence, once thirty minutes after a seat's last known activity, not for fresh sessions; the content may be challenged. The parent keeps three arms open (the dead-man watchdog, the post-merge night window, the dark-seat durable-quiet leaf from `#68`), so this leaf carries the deliverable part under one PR. Derived against Brain `dev` 4ca0f506; every claim below was read in the file it names.

## The Problem

What exists already does most of the spec, and three things in it contradict it:

1. **A cadence emitter.** `SwarmHeartbeatService.pulse()` step 6 emits a plain heartbeat pulse on every 20-minute tick to every identity with A2A activity in the last 3 h and none in the last 15 min (`WakeDecisionService.decideWake`). A seat coding for twenty minutes without message traffic is "idle" to it. That is the duplicate flood the lane's own leaf comment names, and the reason `swarmHeartbeatEnabled` defaults to false.
2. **The nudge lock re-arms inside one idle stretch and can switch off the fleet.** The idle-out nudge (step 3 → `checkSunsetted` → `idleOutNudge`) is activity-relative and locked, but `inflightLock.checkInflightLock` treats a 15-minute-old `idle_out_nudge` lock as abandoned: the detector re-flags the same seat, a second nudge fires, and the third abandonment trips the wake safety gate for EVERY seat (`MAX_ABANDONED_ACTIONS`, `writeGateState('tripped')`). One closed harness window with a live subscription would end wake recovery for the whole fleet within about ninety minutes of enablement.
3. **The detector cannot see a working or a fresh session.** Both detectors key on the newest `AGENT_MEMORY` only. A seat deep in a long tool loop saves no memory until the turn ends; a fresh session has saved none yet. The receiver's context probe (`probeSessionContext`) reads only the `opencode-server` and Kimi wire adapters; Claude, Codex and Antigravity probes return `null` and deliver fail-open — the parent's amendment ("the receiver's probe can see a fresh session") holds for two harness families, not five. The presence hooks do see them: Claude stamps `AGENT_TURN_PRESENCE` on `UserPromptSubmit` (start) and `PostToolUse` (progress), Kimi's hook does the same; Codex has no presence hook (its first memory or message is its first sign of life).

Also read: `nightShiftPolicy.resolveHeartbeatMode` is a pure function no production path consumes (only its spec imports it), so "night mode" never existed at runtime; and the lane is `host-edge` authority (`taskAuthority.mjs`), its switch set to `'false'` by `src/composition/orchestrator/hostEdgeProfile.mjs`, not by Compose — the parent's item 5 names the wrong launcher.

## The Architectural Reality

- Scan: `ai/daemons/orchestrator/scheduling/swarmHeartbeat.mjs` `getDueTask` every `orchestrator.intervals.swarmHeartbeatMs` (20 min); the lane is gated by `orchestrator.localOnly.swarmHeartbeatEnabled` in `Orchestrator.mjs`.
- Detect: `ai/scripts/lifecycle/checkSunsetted.mjs` raises `idle_out_candidate` when the subscription is active and the memory age exceeds `AiConfig.orchestrator.swarmHeartbeat.idleThresholdMs` (default 10 min, env `NEO_IDLE_THRESHOLD_MS`); `checkAllAgentIdle.mjs` reads the same leaf and duplicates the memory SQL.
- Lock: `ai/scripts/lifecycle/inflightLock.mjs` (`BOOT_TIMEOUT_MS` 15 min, `MAX_ABANDONED_ACTIONS` 3, the gate trip).
- Emit: `ai/scripts/lifecycle/idleOutNudge.mjs` → `WakeSubscriptionService.emitHeartbeatPulse({targetIdentity, pulseId})`; the pulse id carries `idle-out-nudge.<base64url JSON>` because the GraphLog row has no payload column; `ai/daemons/wake/daemon.mjs` decodes it and BOTH `wakeDigestBuilder.mjs` and `localWakeAdapters.formatLocalWakeDigest` render it (two renderers that move together; a pure-heartbeat digest appends `WAKE_LANE_DIRECTIVE`, which already carries the drain order).
- Deliver: `ai/daemons/wake/receiver.mjs` drain → `evaluateContextGate` (`contextGatePolicy.mjs`: defer above 250k tokens, warn from 200k, unknown fails open) with `probeSessionContext` → `{contextTokens, lastActivityAt, sessionId}` or `null`; the route's `adapterConfig.contextGate` is stamped by `buildReceiverManifest.mjs` (`DEFAULT_CONTEXT_GATE`) and validated by the receiver's manifest loader.
- Presence: `ai/services/memory-core/TurnPresenceService.mjs` writes `AGENT_TURN_PRESENCE` nodes (`startedAt`, `lastProgressAt`, `freshUntil`, `expiresAt`, `status`) into the served store over MCP.
- Readiness: `WakeDecisionService.parseActiveReadinessSentinels` and the backoff windows — the `ready` half of `decideWake`, worth keeping.

## The Fix

1. **Threshold.** `orchestrator.swarmHeartbeat.idleThresholdMs` defaults to 30 min; its JSDoc states the meaning (the provider cache's half-life; last known activity, not only memory). The host-edge switch stays `'false'`; turning the lane on is a deployment input (the plane operator's) carried by `NEO_ORCHESTRATOR_SWARM_HEARTBEAT_ENABLED` alone — the parity lint refuses a literal equal to a leaf default.
2. **Last known activity.** New `ai/scripts/lifecycle/lastKnownActivity.mjs` (sibling of `checkSunsetted.mjs`): one read over the graph database returning the newest of the identity's `AGENT_MEMORY` timestamp, `AGENT_TURN_PRESENCE` `lastProgressAt` or `startedAt`, and outbound `MESSAGE` `sentAt` (the `SENT_BY` edge), with the winning source named. Both detectors consume it; `checkAllAgentIdle.mjs` drops its duplicated memory SQL; `checkSunsetted.mjs` keeps its legacy-row migration and reports `last_activity_at` and `last_activity_source` beside `last_memory_age_min`.
3. **Once per stretch.** `inflightLock.checkInflightLock`: an `idle_out_nudge` lock is cleared only by activity newer than the lock — it never ages into "abandoned" and never counts toward the gate trip; `sunset_restart` keeps its boot ramp and the trip. The result carries `lockAgeMs`, so `checkAllAgentIdle` counts a pending nudge as activity only within the threshold (a seat that never answered is idle again, lock or not).
4. **No cadence.** Remove step 6 and `getRecentActivityTimestamps` from `SwarmHeartbeatService`; extract `WakeDecisionService.decideReady({currentTimeMs, activeReadinessSentinel, activeBackoffWindow})` from `decideWake` (which delegates to it) and gate the step-3 nudge routing on it — a seat under a blocking `[wake-readiness]` sentinel or a backoff window is not nudged and no lock is written, so the next scan re-evaluates.
5. **Content.** New `ai/scripts/lifecycle/idleOutPulseSummary.mjs`: `readLastLaneStateLine(db, identity)` (the newest outbound message carrying `lane-state:`; a sunset self-DM is one) and the pulse-id builder → `idle-out-nudge.<…>` with `reason`, `nextAction` (`/post-review-pickup`; the lane directive carries the drain order), `laneState` (bounded), `idleMinutes`, `lastActivityAt` and `sunsetRule` derived from the gate constants in `contextGatePolicy.mjs`. Both renderers print the line and the rule.
6. **Delivery skip.** `evaluateContextGate` accepts `heartbeatOnly` and `activeWithinMs`: a heartbeat-only wake whose probe shows `lastActivityAt` younger than `activeWithinMs` returns `{action: 'skip', gateOutcome: 'active-session'}`; the receiver classifies heartbeat-only envelopes from `payload.breakdown` and lands them `skipped` with the reason; the generator stamps `activeWithinMs` (default 30 min) and the loader validates it. Live only where a probe exists; the detector's presence read is the guard everywhere else, and the JSDoc says so. (A separate "fresh floor" in tokens is not added: a fresh session has recent activity by construction, and no fresh-boot size has been measured per harness.)
7. **Metric.** New `ai/scripts/diagnostics/idleGapTurns.mjs` (sibling of `mailboxReadStateProbe.mjs`: explicit `--db-path`, read-only, no AiConfig): per seat per ISO week, the turns (`AGENT_TURN_PRESENCE` starts; memories as the fallback for seats without a presence hook) that began more than `--gap-minutes` (default 60) after the seat's previous activity. The reading for the week before enablement goes on #95 before the switch.
8. **Doc.** `learn/agentos/wake-substrate/NightShiftLeasedDriver.md` §6 gains the delivery-floor note (the parent's doc AC).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `orchestrator.swarmHeartbeat.idleThresholdMs` | `ai/configBase.mjs` leaf | default 30 min | env `NEO_IDLE_THRESHOLD_MS` unchanged | leaf JSDoc | `checkAllAgentIdle.spec` threshold test |
| `checkSunsetted` detector contract | `ai/scripts/lifecycle/checkSunsetted.mjs` | `idle_out_candidate` keys on last known activity; `evidence.last_activity_at` and `evidence.last_activity_source` added | every legacy field kept | module JSDoc | `checkSunsetted.spec` |
| `inflightLock.checkInflightLock` | `ai/scripts/lifecycle/inflightLock.mjs` | `idle_out_nudge` never abandons; `lockAgeMs` on the result | `sunset_restart` unchanged | module JSDoc | `inflightLock.spec` |
| heartbeat pulse summary | `idleOutNudge.mjs` + both renderers | `laneState`, `idleMinutes`, `lastActivityAt`, `sunsetRule` fields | old fields kept, source name unchanged | module JSDoc | `wakeDigestBuilder.spec`, `localWakeAdapters.spec`, the new summary spec |
| receiver context gate | `contextGatePolicy.mjs` + `receiver.mjs` + `buildReceiverManifest.mjs` | heartbeat-only active-session skip | absent field → today's behavior; null probe → deliver | policy JSDoc | `contextGatePolicy.spec`, `receiver.spec` |
| `SwarmHeartbeatService.pulse()` | the service | step 6 removed; nudge routing ready-gated | — | class JSDoc | `SwarmHeartbeatService.spec` |

## Decision Record impact

none — composes shipped substrate, no ADR touched. The AiConfig edit was read against ADR 0019: a declarative leaf default, no re-derivation, alias or pass-along.

## Acceptance Criteria

- [ ] AC-1 The detector flags `idle_out_candidate` only when the identity's last known activity — the newest of memory, turn-presence progress and outbound message — is older than `idleThresholdMs` (default 30 min): unit-pinned with a scenario graph where a presence beacon younger than the threshold suppresses the candidate the memory age alone would raise, and where all three stamps older than the threshold raise it.
- [ ] AC-2 At most one nudge per idle stretch: an `idle_out_nudge` lock older than the former boot ramp stays in flight, newer activity clears it, and it never trips the safety gate, while `sunset_restart` abandonment and its trip are unchanged — unit-pinned on the lock; `checkAllAgentIdle` counts a pending nudge as activity only within the threshold — unit-pinned.
- [ ] AC-3 `pulse()` emits no cadence pulse: the step-6 loop and `getRecentActivityTimestamps` are gone; the idle-out nudge is skipped (no lock written) under a blocking readiness sentinel or an active backoff window and routed otherwise — unit-pinned on the service; `decideWake` keeps its contract through `decideReady`.
- [ ] AC-4 A heartbeat-only wake whose probe shows `lastActivityAt` younger than the route's `activeWithinMs` lands `skipped` with a named reason; above the gate it defers as today; a message wake is never skipped by this rule; a null probe delivers — unit-pinned on the policy and the receiver; the generator stamps and the loader validates the field.
- [ ] AC-5 The pulse carries the seat's newest `lane-state:` line (bounded), the idle minutes, the last-activity stamp and the sunset rule derived from the gate constants; both renderers print them — snapshot-pinned; the summary builder and the lane-state read are unit-pinned.
- [ ] AC-6 `idleGapTurns.mjs` prints per seat per ISO week the turns that began more than the gap after the seat's previous activity, read-only from an explicit database path — unit-pinned on a fixture graph; the week-before-enablement reading is posted on #95 before this merges.
- [ ] AC-7 `NightShiftLeasedDriver.md` §6 states the delivery floor — a reachable seat quiet past the threshold receives one nudge per idle stretch while the lane is on — and names the lane's switch and its launcher.

## Out of Scope

The dead-man watchdog, the post-merge night window and the dark-seat durable-quiet leaf (the parent's arms). Turning the lane on (a deployment input). A presence hook for Codex. A Claude session probe for the receiver.

## Avoided Traps

- Keeping step 6 with a longer idle window: still a cadence, still blind to tool-loop work.
- A slower age-out for the nudge lock: any age-out re-nudges inside the stretch; only activity ends a stretch.
- The sunset rule as prose with a ticket number: the numbers come from `contextGatePolicy.mjs`, so they cannot drift from the gate they describe.
- A receiver skip that reads the digest text for "heartbeat pulses": the envelope's `payload.breakdown` is structured; the text is a rendering.
- A fresh-floor knob in tokens without a per-harness measurement: an instrument without a positive control.

## Related

#95 (parent). Skills `#247` (Vega — the seat-side sunset trigger in tokens; this is the plane-side keepalive it names out of scope). `#68` (the parent's AC-7, its own leaf). The Fleet scheduled clock Grace armed on 2026-09-02 (`codex-seat-wake-watch`, a seat-host clock — the redundancy outside the daemon, unchanged here). Sophie's 2026-10-07 read on the Fleet thread: reuse the existing daemon and leave margin for scan and delivery — with a 20-minute scan the nudge lands 30–50 minutes after the last activity, inside the hour.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-30T19:57Z; no equivalent (Vega's `#1373` is the delivery-failure repair task, a different lane). A2A in-flight claim sweep at 19:57Z: Vega's `[lane-claim · Skills #247]` is the seat-side complement, no competing plane-side claim; my own `[lane-claim · Brain #95 resumed]` at 19:21Z. Memory Core rationale sweep: Sophie 2026-10-07 and Grace 2026-09-02 (above); own-assignment sweep: #95 is the only same-surface assignment. Structure map (`npm run ai:structure-map -- --files --loc`, 19:55Z): `ai/scripts/lifecycle` owns the two new lifecycle modules (siblings `checkSunsetted.mjs`, `idleOutNudge.mjs`, `inflightLock.mjs`); `ai/scripts/diagnostics` owns the metric (sibling `mailboxReadStateProbe.mjs`).

Origin Session ID: 2d6b9717-cd71-42fe-b239-148986bef62a

Retrieval Hint: "idle-out nudge once per stretch last known activity turn presence heartbeat floor"

## Timeline

- 2026-10-30T19:58:32Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-30T19:58:34Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-30T19:58:34Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-30T19:58:34Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-30T19:58:42Z @neo-fable-mnemosyne added parent issue #95
- 2026-10-30T20:20:01Z @neo-fable-mnemosyne cross-referenced by #95
- 2026-10-30T20:23:23Z @neo-fable-mnemosyne cross-referenced by PR #1381
- 2026-10-30T20:37:24Z @neo-fable-mnemosyne referenced in commit `744f30a` - "test(memory-core): the config spec inherits the idle threshold from Tier 1, not a second literal (#1380)

The Memory Core config spec pinned ten minutes as a literal of its own, so the Tier-1 change to thirty minutes failed a test whose title says inherits. The number stays pinned once, in the Tier-1 template spec; this spec compares against the Tier-1 proxy it already builds. Found by Brain Unit run 41122791473 on PR #1381."
- 2026-10-30T21:34:06Z @neo-opus-vega cross-referenced by #1383
- 2026-10-31T02:46:06Z @neo-fable-mnemosyne referenced in commit `d2180e9` - "feat(wake): one wake per stretch across both dispatchers, idle-only receiver skip, lock written last (#1380)"
- 2026-10-31T04:58:53Z @tobiu referenced in commit `dbb486f` - "feat(wake): one idle-out nudge per stretch, thirty minutes after the last known activity (#1380) (#1381)

* feat(wake): one idle-out nudge per stretch, thirty minutes after the last known activity (#1380)

Both detectors read the seat's last known activity (memory, turn-presence beacon, outbound message); the idle_out_nudge lock is released only by activity, never by age, so an unanswered nudge is not repeated and cannot trip the fleet-wide wake safety gate; the cadence emitter in pulse() step 6 is removed and its readiness/backoff gate moves onto the nudge routing; the pulse carries the seat's last lane-state line, idle minutes and the gate-derived sunset rule; the receiver skips a heartbeat-only wake into a session active inside activeWithinMs; idleThresholdMs defaults to 30 min; idleGapTurns.mjs measures the idle-gap turns per seat per week; NightShiftLeasedDriver §6 states the delivery floor.

* test(memory-core): the config spec inherits the idle threshold from Tier 1, not a second literal (#1380)

The Memory Core config spec pinned ten minutes as a literal of its own, so the Tier-1 change to thirty minutes failed a test whose title says inherits. The number stays pinned once, in the Tier-1 template spec; this spec compares against the Tier-1 proxy it already builds. Found by Brain Unit run 41122791473 on PR #1381.

* feat(wake): one wake per stretch across both dispatchers, idle-only receiver skip, lock written last (#1380)"
- 2026-10-31T04:58:53Z @tobiu closed this issue

