---
id: 1742
title: Dispatch urgent A2A without waiting behind ordinary wake batches
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-15T02:25:56Z'
updatedAt: '2026-11-15T16:39:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1742'
author: neo-gpt-emmy
commentsCount: 1
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
closedAt: '2026-11-15T16:39:11Z'
---
# Dispatch urgent A2A without waiting behind ordinary wake batches

## Context

Design authority: high-priority, wake-eligible A2A should get through while a peer works; routine messages can wait. The incident behind #1741 involved a normal notification. This ticket covers the earlier scheduling stages that would delay a genuinely high-priority event even with a working in-turn adapter.

## The Problem

The digest builder preserves the highest surviving priority. However, `CoalescingEngineService` computes a subscription's flush delay without a priority argument, and serializes against prior dispatch. The defaults are a 150-second coalescing window, 120-second refractory interval and 300-second first-event cap. That cap does not bound time spent waiting for a consumer. Receiver records are tried in acceptance order. `stagePrimaryWake` uses one pointer per seat and yields to any earlier record without a delivery receipt, regardless of its priority. Relabeling a digest as high at render time cannot fix those waits.

## The Fix

Expedite eligible high mail through batching and scheduling and keep ordinary records intact. Determine urgency from authenticated, fresh source events after respecting the existing read/suppression rules. Coalesce repeated high signals without letting normal traffic repeatedly extend the urgent deadline. Distinguish a record that has provably never been sent from one whose native delivery is uncertain. If an older normal marker has already been queued, a distinct high record needs independent correlation so the active-turn consumer in #1741 can receive it. Do not evict the old claim or double-send the high record.

| Surface | Authority | Behavior | Evidence |
| --- | --- | --- | --- |
| Coalescing and throttle | Existing signed source priorities and wake policy | High becomes dispatchable at the next scheduling opportunity without an additional normal debounce/refractory delay | Fake-clock controls for normal, high, a suppressed/read high, a normal flood and repeated high |
| Per-seat pending order | Durable receiver records and per-event claims | Unsent normal cannot monopolize the urgent slot; an ambiguous normal remains independently reconcilable | Normal queued then high, crash/restart, duplicate input, two seats and a handoff |
| Delivery reporting | Receiver, trusted recipient, mailbox | Distinguish waiting for policy, chat boundary, and receipt. Report event-to-host and host-to-recipient latency separately | A host dispatch acknowledgment alone does not settle the user-visible delivery |

## Acceptance Criteria

- [ ] A high event arriving during an ordinary coalescing window or refractory interval becomes eligible promptly in a deterministic clock test. An explicitly quiet, already-read or stale high event does not interrupt.
- [ ] Exercise one older normal record both before native send and after queue acceptance without a model-side receipt. The later high record can be offered to its intended recipient once, while the normal record and its deduplication history are preserved. No GUI fallback.
- [ ] Under sustained priority traffic, bounded batching and eventual normal delivery after the burst hold. Repeated/ambiguous sends do not mint duplicate turns or discard mailbox events.
- [ ] Publish one measured high-priority journey with stage timings and the actual delivered outcome. A missing active-turn adapter stays a named dependency rather than a false latency success.

Out of scope: changing producers’ default priority or inventing another priority level. The Desktop active-turn operation and recipient proof belong to #1741.
Decision Record impact: aligned with mailbox-first signed wake custody. Related: #1741 · #1628 · #95 · #1706. A standing delivery-policy change needs an independent review and signed end-to-end validation, not a newcomer assignment.
Live latest-open + closed sweep: newest 20 Brain issues in the red backlog; priority, spool and coalescing searches plus recent 30 A2A rows showed no independent in-flight equivalent. #1741 owns the separate recipient boundary. 
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "high priority wake refractory coalescing blocked by earlier normal pointer"

## Timeline

- 2026-11-15T02:25:57Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T02:25:57Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T02:25:58Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T08:35:27Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-11-15T08:35:30Z

Lead-role active: planning, design dialogue, and peer coordination count as execution; suspend Auto Mode velocity bias until an exit condition is met.

### Ticket Intake (`valid-as-written`)

- **Ticket age & bot-state:** `createdAt: 2026-11-15T02:25:56Z`, `updatedAt: 2026-11-15T02:25:56Z` (`pre-stale`, `stale: false`, `no auto close: false`).
- **Prescription checked:**
  - `ai/services/memory-core/wakeCoalescePolicy.mjs` + `ai/services/memory-core/CoalescingEngineService.mjs` — owns priority-aware coalescing/refractory scheduling, bounded urgent burst batching, and freshness/read-state urgency gating without normal arrivals extending an urgent deadline.
  - `ai/daemons/wake/receiver.mjs` + `ai/daemons/wake/primaryWakeSpool.mjs` + `ai/daemons/wake/primarySessionWake.mjs` + `ai/daemons/wake/fleetPrimarySessionWake.mjs` (+ `ai/daemons/wake/antigravityWakeHook.mjs`) — owns urgent pending drain ordering, preemption of provably-unsent normal pointers, and an independent urgent correlation slot (`urgentInflight` / `<seat>.urgent.request.json`) that neither evicts an ambiguous queued normal `inflight` nor falls back to GUI typing, while surfacing a named active-turn adapter dependency when the active-turn adapter (`#1741`) is unavailable on a busy root.
  - `ai/services/memory-core/wakeDeliveryProjection.mjs` — owns stage-state (`waiting-for-policy`, `waiting-for-chat-boundary`, `waiting-for-receipt`, `delivered`) and separate `eventToHostMs` / `hostToRecipientMs` latency reporting so host HTTP/queue ack alone never settles user-visible delivery.
- **Currency / successor-risk & ADR check:** `ADR successor-risk: no-adr-impact` (aligned with `neomjs/neo-agent-brain#adr:phase3-wake-substrate-standards-alignment` and `ADR-0019` read in full; `#1741` owns the separate Codex Desktop `turn/steer` boundary; `0` open/closed PRs on `#1742`; Contract Ledger verified against current `dev`).
- **Core-Idiom Pre-Flight (`§1.9.6`):** Read `src/Neo.mjs`, `src/core/Base.mjs`, and `src/state/Provider.mjs` alongside `ADR-0019`.

Claiming `#1742` and branching `agent/1742-urgent-wake-dispatch` from `origin/dev`.

- 2026-11-15T09:14:31Z @neo-gemini-julia cross-referenced by PR #1759
- 2026-11-15T15:14:30Z @neo-gemini-julia referenced in commit `338df76` - "feat(wake): expedite urgent A2A wakes across coalescing and native spool (#1742)"
- 2026-11-15T15:14:31Z @neo-gemini-julia referenced in commit `b9250cd` - "fix(wake): bound legacy urgency, wire stage split, and declare urgent leaves (#1742)"
- 2026-11-15T16:39:11Z @tobiu referenced in commit `ed48a70` - "feat(wake): expedite urgent A2A wakes across coalescing and native spool (#1742) (#1759)

* feat(wake): expedite urgent A2A wakes across coalescing and native spool (#1742)

* fix(wake): bound legacy urgency, wire stage split, and declare urgent leaves (#1742)"
- 2026-11-15T16:39:12Z @tobiu closed this issue

