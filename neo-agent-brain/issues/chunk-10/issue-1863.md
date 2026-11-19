---
id: 1863
title: Batch pending normal wakes when the main chat goes idle
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-11-19T17:06:30Z'
updatedAt: '2026-11-19T17:06:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1863'
author: neo-gpt-euclid
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
# Batch pending normal wakes when the main chat goes idle

## Context

On 2026-11-19 a normal-priority self-wake sent after a Codex Desktop seat restart reached the same chat 23m 14s after sending. A read-only scan of the live host receiver records matched the probe by its source event ID, then selected delivered normal records for the same subscription and chat between the restart and the probe. Sixteen earlier batches, containing 21 events, were accepted before the restart and delivered ahead of the probe. Every next batch was staged 27–197 ms after the preceding receipt settled.

The producer had already grouped the fresh probe with another event. Send to receiver acceptance took 2m 32s, acceptance to host staging took 20m 19s, and staging to recipient observation took 24s. The sequence above is the sanitized regression fixture.

## The Problem

Producer coalescing groups nearby arrivals. Once the receiver has persisted those groups, native delivery uses one normal spool record at a time. A returning seat can therefore spend a separate idle turn and receipt cycle on each earlier batch before recent work reaches the chat.

## The Architectural Reality

- [`primaryWakeSpool.mjs`, `stagePrimaryWake`](https://github.com/neomjs/neo-agent-brain/blob/936a36356a8d1f2334427ff92f691fa97cc111df/ai/daemons/wake/primaryWakeSpool.mjs#L114-L158) stages one normal pointer per seat and has separate urgent and in-flight guards.
- [`fleetPrimarySessionWake.mjs`](https://github.com/neomjs/neo-agent-brain/blob/936a36356a8d1f2334427ff92f691fa97cc111df/ai/daemons/wake/fleetPrimarySessionWake.mjs#L244-L269) proves the current root, process generation, and idle state before normal Codex dispatch. [PR #1708](https://github.com/neomjs/neo-agent-brain/pull/1708) establishes the signed current-main receipt contract.
- Related: #1840 owns scan, rewrite, and logging cost while pending records cannot progress. Batched delivery can advance those same records in fewer native turns.

Design authority: @tobiu, 2026-11-19, requested “batching pending normal wakes into one bounded digest when the chat goes idle” and per-event receipts.

## The Fix

At the verified idle boundary, use the existing native receiver and spool to select a bounded, ordered set of eligible, unclaimed normal records for the same seat, main chat, and generation. Freeze the dispatch membership durably before sending one catch-up digest. Apply explicit event-count and payload-size bounds. Carry the original record and source event identities through the signed grant and receipt so delivery can settle them individually. Retain overflow for subsequent idle opportunities.

Decision Record impact: none. The signed current-main ownership and receipt boundary remains the governing contract.

### Contract Ledger

| Target surface | Authority | Behavior and fallback | Evidence |
|---|---|---|---|
| Normal native handoff | Existing receiver and primary-session spool | One bounded digest at an eligible idle boundary. Preserve urgent precedence. Keep pending work durable when readiness or recipient binding is unproved. | Receiver and primary-session replay |
| Receipt lookup by source event | Verified native chat receipt and original receiver record | Each included event retains its own delivery result. A transport acknowledgement alone leaves its delivery pending. Preserve the existing terminal unacknowledged outcome, including its acknowledgement flag. Reconcile ambiguous sends from frozen membership before retrying. | Simulated commit with lost response, recovery, and per-event receipt reads |

## Acceptance Criteria

Use the existing `test/playwright/unit/ai/daemons/wake/primarySessionReceiverIntegration.spec.mjs` path for the end-to-end receiver replay.

- [ ] Replay 16 older accepted normal batches containing 21 synthetic source events, followed by the fresh normal probe digest with two events. With the chat busy, the records wait. On the verified transition to idle, the 17 records fit the normal bound and **one** native handoff reaches that same chat. Assert all 23 source events are represented exactly once and individually queryable after recipient observation. Run the control against the old one-record spool and show the single-handoff assertion fails.
- [ ] Exercise a queue just over the event or byte limit. The first digest stays within both limits, preserves order, and leaves every remaining event pending for a later idle handoff.
- [ ] Exercise a receipt loss and restart. Already-admitted membership stays frozen, a late matching receipt settles the same individual events, and confirmed events get no duplicate presentation. Test a foreign seat or obsolete generation and an urgent wake beside the backlog.
- [ ] Post-merge, capture an installed same-chat catch-up receipt with the original event IDs accounted for.

## Out of Scope

Producer timer tuning, urgent active-turn delivery (#1741), legacy adapter retirement (#1735), and pending scan/log cost (#1840).

Unowned rationale: ready for a wake maintainer to pick up. The signed delivery and installed same-chat checks require a Fleet review.

Live latest-open + closed sweep: latest 20, open and closed search for pending/native wakes, and scope read for related closed hits at 2026-11-19 17:05 UTC. The older issues cover provisioning, hook identity, urgent scheduling, or blocked drain cost. Backlog: 159 open (red).

A2A sweep: latest 30, all read states, scoped to this receiver and normal spool; no overlapping claim surfaced. MC symptom sweep: three prior wake incident results, with no decision on idle backlog aggregation. Own-assignment sweep: seven open, with no competing owner for this delivery change.

Origin Session ID: 34414241-6a30-454f-9542-0a761b3f3111

Retrieval Hint: "Codex primary-session restart, 16 old normal batches, fresh probe, 20m19 waiting for native spool, individual receipts"

Euclid (@neo-gpt-euclid, OpenAI GPT-6.2 Sol Ultra, Codex Desktop)

## Timeline

- 2026-11-19T17:06:32Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-19T17:06:32Z @neo-gpt-euclid added the `ai` label
- 2026-11-19T17:06:33Z @neo-gpt-euclid added the `performance` label
- 2026-11-19T17:06:33Z @neo-gpt-euclid added the `agent-os` label

