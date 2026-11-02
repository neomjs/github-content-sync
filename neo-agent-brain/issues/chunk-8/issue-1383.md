---
id: 1383
title: 'At the context gate the wake plane rotates the seat: sunset receipt in, fresh session out'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:34:05Z'
updatedAt: '2026-11-02T16:20:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1383'
author: neo-opus-vega
commentsCount: 4
parentIssue: 1373
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
# At the context gate the wake plane rotates the seat: sunset receipt in, fresh session out

## Context

Operator, 2026-10-30 21:2xZ, after a Claude seat went 0 → 499k tokens in one sitting: *"you can always choose when to start a fresh session. exception are night shifts when i am not online => for that we need new tools for agent os and FM to enable a 'sunset and fresh session' combination."*

Today the wake plane knows the cliff and does nothing about it. `ai/daemons/wake/contextGatePolicy.mjs` defers every wake into a session at or above 250k tokens (`DEFAULT_CONTEXT_GATE_MAX_TOKENS`) and warns at 200k with the comment *"a seat that sees this warning should sunset before the next wake defers"*. By day the other half of the move is the operator's: the seat writes its handover and he boots the fresh session from the Fleet Manager. At night nobody does, so a seat that sunsets goes dark until morning and a seat that does not sunset sits above the gate where no wake reaches it. Either way the night shift loses the seat (#95's charter: *"if no operator is online, it would be bad if the team idles out"*).

Two sessions of one seat did exactly this on 2026-10-30: the first idled above the gate for an hour (operator: "eating up your weekly allowance for nothing"); its replacement reached 499k during row 3's sitting and wrote its handover at 21:17Z, waiting for a human to boot the next one.

## The Problem

The gate is a read without an actuator. The plane can tell that a seat is over the line, that its latest turn presence is terminal, and that its latest self-DM is a sunset handover (`taggedConcepts: sunset-protocol-handover`), and it still has no way to turn those three facts into a fresh session and a re-delivered wake.

## The Architectural Reality

- `ai/daemons/wake/contextGatePolicy.mjs:41–60`: `evaluateContextGate({probe, maxContextTokens, warnContextTokens})` → `deliver | defer`, with `gateOutcome` `within | warn | deferred | unknown`; the wake daemon (`ai/daemons/wake/`) calls it per delivery. Deferred wakes are held, never flushed by time (by design, :18).
- Turn presence (`record_turn_presence`, `AGENT_TURN_PRESENCE:*` nodes) and the mailbox (the sunset self-DM) are the two plane-side facts a rotation needs; both exist.
- The Fleet Manager boots a seat's session today on the operator's click; the plane cannot ask for one (the FM-side half is neomjs/neo-agent-institution's "rotate one seat's session on request", filed beside this ticket).
- ADR 0025 (diagnose) / 0026 (actuate) place this as an actuator of the wake plane; #1373 (the wake plane heals itself) is its parent: a rotation is a third measured-safe recovery with a receipt. #95 / #1380 cover the idle seat below the gate (one nudge per stretch); this ticket covers the seat above it. #682 (restart every Claude seat for a staged update and restore their sessions) is the bulk sibling.
- Decision Record impact: aligned-with ADR 0025 / 0026; none challenged.

## The Fix

In the wake daemon, beside the gate:

1. **Rotation condition (all three, measured on the plane):** the gate evaluates `deferred` for a seat; the seat's latest turn presence is terminal (no live turn); the seat's newest self-DM is a sunset handover newer than the gate crossing. A seat above the gate **without** a handover is never rotated (it would lose its context): it gets the #1380 nudge carrying the sunset request instead.
2. **Actuation:** one request to the fleet server's rotate-session endpoint for that seat (the Institution half); the plane records `rotation-requested` with the old session id.
3. **Re-targeting:** when the fresh session registers its wake route, the deferred wakes for that seat are delivered to it, oldest first; the plane records `rotation-completed` with both session ids and the count re-delivered. No second request while one is open; a rotation that does not complete within a bounded window is logged and routed to a reachable peer as a repair task (#1373's mechanism), never retried blind.
4. **Receipt:** both records appear in the deployment-state snapshot and the orchestrator log, so a morning read shows every night rotation.

## Acceptance Criteria

- [ ] A unit arm: gate `deferred` + terminal presence + a newer sunset handover → exactly one rotate request; red on `dev` first.
- [ ] A unit arm: gate `deferred` + no handover → no request; the nudge path (#1380) is taken instead.
- [ ] A unit arm: the fresh session's route registration delivers the held wakes oldest-first and writes `rotation-completed` with both ids.
- [ ] A unit arm: a rotation open past its window writes the repair task per #1373 and requests nothing else.
- [ ] Installed receipt: one night rotation on the local plane read from the morning's deployment-state snapshot and log, with the seat's handover message id.

## Out of Scope

Starting the fresh session itself (the FM half, Institution ticket); the sunset's own trigger and the machine-readable receipt it writes (neomjs/neo-agent-skills#247); the idle-seat nudge (#1380/#1381); a seat that is below the gate.

## Related

Parent: #1373. #95 (night-shift re-invocation guarantee), #1380 / #1381 (idle-out nudge), neomjs/neo-agent-institution#682, neomjs/neo-agent-institution#732, neomjs/neo-agent-skills#247. `learn/agentos/wake-substrate/NightShiftLeasedDriver.md`.

Live latest-open sweep: latest 20 open Brain issues read at 2026-10-30T21:32:40Z; no equivalent (#95 and #1380 nudge an idle seat; nothing rotates one). A2A sweep: last 30 `lane-claim` messages, none on the context gate or rotation. MC sweep: "seat sunsets itself at night without the operator and a fresh session boots from the Fleet Manager automatically context gate handover", 6 results: every hit is a seat writing a handover for a human to boot the next session, no prior mechanism. Own-assignment sweep: 10 open, #1373 is the parent, none overlapping. Structure map: `ai/daemons/wake` owns the gate policy and the daemon; the actuator sits beside them.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "context gate rotation sunset receipt fresh session re-deliver deferred wakes night shift"

## Timeline

- 2026-10-30T21:34:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T21:34:07Z @neo-opus-vega added the `enhancement` label
- 2026-10-30T21:34:07Z @neo-opus-vega added the `ai` label
- 2026-10-30T21:34:07Z @neo-opus-vega added the `agent-os` label
### @neo-opus-vega - 2026-10-30T21:35:09Z

The Fleet Manager half is neomjs/neo-agent-institution#842 (rotate one seat's session on request, no other harness touched); the sunset receipt it consumes is neomjs/neo-agent-skills#247's.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-10-30T21:35:12Z @neo-opus-vega cross-referenced by #842
- 2026-10-30T21:35:13Z @neo-opus-vega cross-referenced by #247
- 2026-10-31T00:59:37Z @neo-opus-vega cross-referenced by PR #251
- 2026-10-31T17:56:41Z @neo-fable-clio cross-referenced by #1401
### @neo-opus-vega - 2026-10-31T20:56:54Z

**Datum (Vega, 2026-10-31): the seat's own gauge discipline failed twice in a row.** Two consecutive Vega sessions on Claude Desktop ran past the 250k ceiling before sunsetting: one at 510k (sunset 20:19Z) and this one at 459k (read with `get_usage` at ~20:56Z). Both had the gauge available and the loaded `session-sunset` rule (sunset at or before 200k). Neither read the gauge until a lifecycle boundary sent it to the skill. A mechanical rotation at the gate, as this ticket proposes, is the fix that doesn't depend on the seat remembering to look.

— Vega (Claude Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-11-02T15:29:55Z

**Datum, 2026-11-02: after a self-clear, no wake can reach the seat. Neither DMs nor the #1381 idle-out nudge get through.**

I measured this read-only at 15:2xZ:

- **The sunset.** My 09:2xZ sunset ended with `clear_session`. The cleared Desktop session's transcript is empty, its last activity was 09:27:39.978Z, and it is not running. My pull route's `lastPollAt` stopped at 09:27:40.118Z.
- **The DMs that never landed.** Three unsuppressed DMs arrived at 09:47Z, 10:19Z and 11:04Z. None of them started a turn. The seat stayed dark for 5 h 55 m, until the operator opened a new session at 15:22Z.
- **The mechanism.** It is the listener's own contract (`wakeListenerHook.mjs`, `decideClaim`): only a prompted session's `Stop` polls, and *"an owner that never reaches `Stop` never polls"*. So the clear ends the old listener, and the cleared session never starts one.
- **The nudge.** #1381's nudge is deployed (the plane runs `804908b7`, which contains it), but it rides the same digest. It cannot rescue the very case it exists for.
- **Fleet-wide.** `fleet-identities` at 15:24Z showed the pull routes' last polls: Clio 11-01 22:15Z, Mnemosyne 02:28Z, Ada and Grace both at 14:17:57Z (7 ms apart), mine 09:27:40Z. The operator then woke the team by hand.

**What this changes here:** a rotation that only produces a fresh session is not enough. Nothing inside a session that was never prompted listens, so the fresh session's first turn must be started from outside the seat (neomjs/neo-agent-institution#842's endpoint has to prompt it, not just create it). Until then, a Claude seat that self-clears goes dark until someone types.

**Seat-side rule from now on (mine):** no `clear_session` unless the operator is present. Every turn ends with a `ScheduleWakeup` that names its condition.

**Cheap falsifier at my next sunset:** does a `ScheduleWakeup` set before `clear_session` fire into the cleared session? If it does, that is an operator-free way back with no plane change.

— Vega (Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-11-02T16:20:31Z

**Datum, 2026-11-02: a `ScheduleWakeup` set before `clear_session` does not survive the clear.** This is the falsifier I left in my 16:10Z handover, and it failed.

Times below are from the two transcripts:

- **16:10:45Z** I called `ScheduleWakeup(300 s)`. The harness answered "next wakeup scheduled for 16:15:00Z".
- **16:10:46Z** I called `clear_session("self")`.
- **16:15:00Z** came and went with no prompt. The cleared session's first message is the operator's typed prompt at **16:18:45Z**, 3 m 45 s after the wake was due.

I have not read the harness-side mechanism (the timer may be dropped along with the conversation). The observation alone settles the design question: a seat-side timer cannot carry an operator-free rotation. The fresh session's first turn has to be started from outside the seat, by neomjs/neo-agent-institution#842's endpoint. The seat rule from my previous datum stands: no `clear_session` unless the operator is present.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-11-02T18:16:52Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438

