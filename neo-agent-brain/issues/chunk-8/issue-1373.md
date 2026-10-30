---
id: 1373
title: 'The wake plane heals itself: a sustained delivery failure routes one repair task to a reachable peer, and the two measured-safe recoveries run with a receipt'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-30T02:43:26Z'
updatedAt: '2026-10-30T02:59:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1373'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
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
# The wake plane heals itself: a sustained delivery failure routes one repair task to a reachable peer, and the two measured-safe recoveries run with a receipt

## Context

Wake delivery has been repaired by hand six times in four weeks, each time after a peer or the operator noticed the silence: a missing `agentIdentity` on the envelope (2026-09-26, #529), a host running stale code after #838 (10-04), a seat's focus loss on `osascript` (10-08), a wake daemon deadlocked in `uv__threadpool_cleanup` on exit and killed by hand so launchd would respawn it (Maryam, 10-24), a route withdrawn after HTTP timeouts and resumed by its owner (Emmy, 10-24), subscriptions reset by hand to clear failure counters (Julia, 10-24), and tonight the Codex activation guard reading a rebound System Events PID (Emmy, #936, 10-30). After tonight's repair the operator asked Emmy how it worked and whether tickets should automate recovery, because wakes keep breaking; Emmy routed the question to the owner of #503 and #30 (A2A 0100f30a, 02:40Z): "the operator should not have to discover another silent multi-day failure."

## The Problem

The wake plane has the detect half of an immune system and no act half. Since #503 the delivery projection carries each subscription's `consecutiveFailures` and the receiver's own `lastOutcomeReason`, and `who_is_online` says *present and undeliverable* instead of healthy. Nothing consumes that signal: a route can fail every dispatch for days with the diagnosis sitting on the health surface, and the repair happens when a human reads it. The container plane solved this shape in ADR-0025 (diagnose) and ADR-0026 (a bounded, reversible, rate-limited actuator that consumes the diagnosis or records it); the wake plane, a host launchd service (`deploy/host/com.neomjs.agent-os-wake.plist`) outside `ProcessSupervisorService`'s reach, has neither a consumer nor an actuator.

## The Architectural Reality

- Detect exists: `ai/services/memory-core/wakeDeliveryProjection.mjs` (`consecutiveFailures`, `lastOutcomeReason`, #503 AC-3/AC-4); `who_is_online` projects reachable vs undeliverable with the receiver's reason.
- Fail-closed pieces exist inside the daemon: `buildReceiverManifest.mjs` withdraws a stale route with a named reason; `receiver.mjs` runs every state call under a step watchdog; `receiverLiveness.mjs` reads the receiver's own liveness. None of them acts on a *sustained* failure of a published route.
- The actuator discipline exists for the container plane: ADR-0026 (one bounded action per diagnosis, persisted anti-thrash state, `record` when the class is outside the action set, a receipt after the action); its B0 tier recycles orchestrator children on a sustained-failure probe (`ProcessSupervisorService`, #13882). The wake daemon is not a child of the orchestrator.
- A task can already be routed to a peer: the mailbox with `taskState` (blocked-task-state), and `who_is_online` names who is reachable right now.
- **Two states a receipt must keep apart (Emmy, A2A 8e61f999):** the receiver's `delivered` proves the dispatch script returned successfully, not that the addressed agent resumed a turn. Tonight both were witnessed separately: `delivered` on the record, and automatic prompts arriving in the Emmy and Sophie chats. A withdrawal, likewise, is either automatic and transient (timeouts) or the owner's intentional pause; the projection does not say which by itself.
- The measured failure classes and their repairs, from the six incidents:
  1. daemon deadlock on exit (`uv__threadpool_cleanup`) → kill the PID, launchd respawns (Maryam 10-24);
  2. route withdrawn after HTTP timeouts → resume the subscription (Emmy 10-24);
  3. `-2700 Target app lost frontmost status` on `osascript` → the Codex side is #936's activation fix (#1372); the Claude side is #30's native delivery; until then a retry with activation;
  4. `-25211` Accessibility consent → no automation possible; the exact dialog goes to the operator (#503 AC-6);
  5. a pre-identity envelope schema (#513) → owner-scoped envelope repair.
- Thresholds are config leaves; the implementer reads ADR-0019 before touching `ai/` config (AGENTS.md gate 10).

## The Fix

A consumer of the existing health surface, shaped like ADR-0026, for the wake plane:

1. **Detector:** a subscription whose `consecutiveFailures` has stayed at or above `N` for longer than `T` (both config leaves) is a diagnosis `{seat, subscriptionId, class, lastOutcomeReason}`; an unreadable projection is `unknown` and fires nothing. No new bookkeeping: it reads what #503 already writes.
2. **Actuator, two classes only:** class 1 (relaunch the deadlocked daemon through launchd) and class 2 (resume the withdrawn route), each one bounded, reversible action under a persisted anti-thrash cap, followed by a **two-state receipt**: `dispatch-restored` when one real dispatch is recorded `delivered` by the receiver (#503 AC-2's standard), and `agent-resumed` only on the addressed seat's own acknowledgement (its next `record_turn_presence` or `add_memory` after the dispatch). Absent the acknowledgement the seat is `dispatch-restored`, never `healed`. Class 2 runs **owner-scoped**: it resumes only a withdrawal the projection records as automatic (timeouts, a stale route rebuilt), never one the owner paused, and it runs under the owner's authority, not the repairer's.
3. **Routing:** a diagnosis outside the action set, or an action whose receipt fails, becomes exactly one `[wake-repair]` task (A2A, `taskState`) to the first *reachable* peer whose route family differs from the failing seat's, deduplicated per failing seat per window, with the diagnosis and the repair recipe for its class attached. The recipient is a repair recipient, not an authority over the route owner's decisions: a recipe that touches the owner's route says so and stops at the owner's scope. The operator is addressed only when no peer is reachable or the class is 4.
4. **Record:** every diagnosis, action and receipt lands on the delivery projection's own ledger so the next reader sees what was tried, not only that the route is red.

## Acceptance Criteria

- [ ] AC-1 The detector fires on a synthetic subscription held at `consecutiveFailures ≥ N` for `> T`, does not fire below either threshold, and records `unknown` without firing on an unreadable projection (unit spec, red-first on each arm).
- [ ] AC-2 Classes 1 and 2 run their action under the anti-thrash cap and record `dispatch-restored` only on a real `delivered` dispatch, never a shape assertion; a second diagnosis inside the cap is recorded, not acted on (unit spec on the envelope; one live arm per class on a disposable subscription).
- [ ] AC-3 A diagnosis outside the action set routes exactly one `[wake-repair]` task to a reachable peer of another route family within `T`, with the diagnosis and the class recipe attached; a second diagnosis for the same seat inside the window routes nothing (unit spec; one live arm where the task lands in a real peer's mailbox).
- [ ] AC-4 The operator is addressed only when no peer is reachable or the class is 4; the message carries the exact dialog or reason (unit spec).
- [ ] AC-5 *(non-vacuity)* A plane with every route delivering routes nothing and acts on nothing across a full window; the detector is neither permanently red nor permanently green.
- [ ] AC-6 The receipt keeps `dispatch-restored` and `agent-resumed` apart: a seat with a `delivered` dispatch and no acknowledgement reads `dispatch-restored`, one with the seat's own turn-presence or memory write after the dispatch reads `agent-resumed`, and nothing reads `healed` without the second (unit spec; one live arm witnessing a real resumed turn).
- [ ] AC-7 Class 2 refuses a withdrawal the owner recorded as intentional and resumes only an automatic one, under the owner's scope (unit spec, red-first: an intentional pause must not be resumed).

## Post-Merge Validation

- [ ] One real recurrence of class 1 or 2 reaches `dispatch-restored` with a receipt, or is routed, before any human notices the silence; the ledger shows the diagnosis, the action and both receipt states.

## Out of Scope

- Native per-session delivery that removes the focus dependence (#30) and the Codex activation path (#936, #1372): this leaf repairs around them until they land.
- The daemon's internal structure (#609) and the cross-harness wake tiers (#129, #130).
- Automating class 3, 4 or 5: their repairs are not measured safe. No focus retries, no permission resets.

## Avoided Traps

- A new watchdog that polls the seats: the signal already exists on the projection; a second reader of the same facts is the drift ADR-0019 forbids.
- An actuator without a receipt: a recovery that reports success by its own exit code is #503's defect one level down.
- A `delivered` dispatch read as a healed seat: the script returned; whether the agent resumed is a second fact with its own witness.
- A repairer resuming what an owner paused: a reachable peer is a recipient, never authorization over the route owner.
- Routing the repair to the broken seat, or to every seat: one task, one reachable peer, another route family, a window.
- Paging the operator first: an operatorless night has no one to page (ADR-0026's 2026-06-26 amendment); the operator is the last recipient, never the detector.

## Related

#503 (the detect half, open, AC-6/AC-7) · #30 (native delivery) · #936 / #1372 (Codex activation) · #95 (presence-aware wake policy) · #513 · #514 · #609 · ADR-0025 · ADR-0026 · A2A 0100f30a (Emmy, the operator's question) · A2A 8e61f999 (Emmy, the two boundaries)

Sweeps: the latest 20 open Brain issues at 2026-10-30T02:41Z and the open issues with "self-healing", "actuator" or "wake" in the title (#118, #50, #936, #130, #129, #95, #514, #513, #88, #549, #609, #30, #146, #69, #125) carry no consumer of the delivery health surface; A2A: Emmy's ask and no competing claim; Memory Core: the six repairs above, every one by hand; own assignments: #503 and #30 (the detect half and the transport), neither owning the act half.

Decision Record impact: none new; the leaf applies ADR-0026's envelope to the wake plane. If the implementer finds the envelope needs a host-plane variant, that is an ADR-0026 amendment, not a new ADR.

Origin Session ID: b8b1a292-24f5-42c2-a898-54f6192b9282
Retrieval Hint: "wake plane self-healing consumer consecutiveFailures diagnosis actuator receipt dispatch-restored agent-resumed owner-scoped resume route repair task reachable peer launchd relaunch"


## Timeline

- 2026-10-30T02:43:27Z @neo-opus-vega added the `enhancement` label
- 2026-10-30T02:43:27Z @neo-opus-vega added the `ai` label
- 2026-10-30T02:43:28Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T02:43:55Z @neo-opus-vega cross-referenced by #503
- 2026-10-30T03:04:58Z @neo-opus-vega cross-referenced by PR #1372

