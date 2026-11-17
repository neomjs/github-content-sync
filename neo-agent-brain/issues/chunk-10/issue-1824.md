---
id: 1824
title: An unredeemed urgent wake holds every ordinary wake behind it
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-11-17T00:03:29Z'
updatedAt: '2026-11-17T06:19:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1824'
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
closedAt: '2026-11-17T06:19:50Z'
---
# An unredeemed urgent wake holds every ordinary wake behind it

> Reshaped 2026-11-17 03:4xZ by the prescription check, before any code: the first Fix withdrew the grant and let the record re-send into the same session. When the hook is what failed, every re-send fails the same way. The Fix below settles the attempt on evidence the receipt reader already parses, and re-sends nothing. The first Fix is kept under Avoided Traps.

## Context

On 2026-11-16 the operator noticed that a review request to @neo-gpt-euclid (middleware-v2 #85, requested at 22:47Z) never woke the seat. Euclid's own reading (A2A `bc1cecef`, 23:58Z) and mine of the receiver's records agree:

- At 09:29:46Z an urgent digest was accepted for Euclid's `primary-session` route (Codex Desktop). Its source was Julia's high-priority broadcast `b76163b2`. The receiver first sent it at 12:16:34Z (`deliveryMode: idle`).
- From then on the primary-session state held `urgentInflight: {status: "unknown"}`. No urgent claim file existed, so the recipient hook never redeemed the nonce; the receiver re-checked it as `native-recipient:receipt-unproven`.
- An ordinary digest carrying the review wakes for middleware-v2 #85 and Skills #414 was accepted at 23:04:36Z and stayed pending behind it.

**How it ended:** Euclid's Codex app restarted at 23:19:05Z. That is the one path that withdraws an unclaimed grant today (a generation change). The receiver re-granted on the new session, and the hook redeemed the new nonce at 00:57:08Z (record `f817d9d4…`, `delivered` at 00:57:38Z); the held ordinary digests followed (Euclid's A2A `c4a8b744`, `7f914c5e`). So no ordinary wake reached the seat for about eleven hours of an unchanged session, and only a restart ended it.

## The Problem

An ordinary delivery waits while any attempt in its seat's other slot is unresolved, and an attempt the hook never redeems stays unresolved until the session changes. Nothing bounds that, and nothing tells the attempt that can still be redeemed apart from the one that cannot.

Why the hook did not redeem is a separate question. Lost Codex hook approval (#1803, fixed by #1805 on 11-16) is the likely cause; it is unverified for this seat.

## The Architectural Reality

- `ai/daemons/wake/primarySessionWake.mjs`, `deliverPrimarySessionWake()`: the per-seat slots `inflight` and `urgentInflight` (#1742, PR #1759). An ordinary record returns `{status: 'pending', reason: 'urgent delivery in progress'}` while `state.urgentInflight` exists. A slot settles on a `consumed` receipt (delivered) or a `not-sent` one (cleared).
- `ai/daemons/wake/codexSessionWake.mjs`, `sendCodexSessionWake()`: the wake is `codex queue --thread <id> --message <text>`. The CLI has no command that lists or cancels a queued message, so a queued wake cannot be taken back.
- `ai/daemons/wake/codexSessionWake.mjs`, `readCodexWakeReceipt()`: scans the session log from the attempt's cursor. It already finds the queue turn (`task_started`, trigger `queue`), the user message carrying the exact wake text, and the hook's acknowledgement (`[NEO_WAKE_RECEIVED:<token>]`) in that turn. It returns `turn-started` with the acknowledgement and `unknown` otherwise. `task_complete` and `turn_aborted` carry their `turn_id`.
- `ai/daemons/wake/fleetPrimarySessionWake.mjs` (about lines 192–212) withdraws an unclaimed grant under the recipient lock, gated on `current.binding.generation !== generation || current.binding.threadId !== next.threadId`. After a withdrawal the pump dispatches the record again with a new nonce (the 00:57Z dispatch above).

Design authority: #1742, "Dispatch urgent A2A without waiting behind ordinary wake batches", makes urgent delivery independent of ordinary delivery. `deliverPrimarySessionWake()`'s JSDoc protects the reverse case only ("distinct urgent records use `urgentInflight` without evicting an ambiguous ordinary attempt"). No record makes ordinary delivery wait without bound behind an attempt nobody can redeem.

## The Fix

Settle an attempt when the session log proves it is final, and re-send nothing:

1. `readCodexWakeReceipt()` returns `{state: 'turn-unacknowledged', threadId, turnId}` when the exact wake text started its queue turn and that turn completed or aborted without the hook's acknowledgement. Such an attempt is final: the interrupt landed (a turn started, so the seat woke and reads its mailbox), the digest did not, and that prompt is never submitted again.
2. `deliverPrimarySessionWake()` settles a slot on that receipt as `delivered` with `acknowledged: false`, under the same lock and only when no admitted claim exists for the nonce. The slot frees, and the next record, urgent or ordinary, dispatches.
3. The receiver keeps `acknowledged: false` on the record, so the route's health (#503) can show a seat that wakes without its digest.
4. A wake text that has not started a turn keeps its slot, as today: it may still drain from the queue and be redeemed.

Scope: Codex idle delivery, the path observed here. Active-turn delivery and Antigravity keep today's behaviour.

## Acceptance Criteria

- [ ] AC-1: `readCodexWakeReceipt()` returns `turn-unacknowledged` for a queue turn that carries the exact wake text and completes or aborts without the acknowledgement; `unknown` while that turn is still running; `turn-started` as today when the acknowledgement is present.
- [ ] AC-2: `deliverPrimarySessionWake()` settles that receipt as delivered with `acknowledged: false` and frees the slot; an ordinary record held behind the urgent one dispatches on the next cycle.
- [ ] AC-3: an attempt whose claim was admitted is never settled as unacknowledged, and a hook that runs after the settlement finds no matching grant and discloses nothing.
- [ ] AC-4: unit specs cover AC-1 to AC-3 through the modules' injected seams, red first on each arm.
- [ ] AC-5 (post-merge): the next real unacknowledged wake settles without a session change; its record shows `acknowledged: false`, and an ordinary digest behind it is recorded `delivered`.

## Out of Scope

- Why a hook does not redeem (#1803 / #1805) and delivery into an active turn (#1741).
- The health projection that called the route deliverable (#503); this ticket only records `acknowledged: false` for it.
- A queued wake that never starts a turn: it keeps its slot, and #503 is where it becomes visible.

## Avoided Traps

- **Withdrawing on an unchanged session after a turn or an age bound** (this ticket's first Fix). The pump re-sends the record with a new nonce into the same session. If the hook is what failed, each re-send fails the same way: under a turn trigger, every bare wake turn would trigger the next one; under an age bound, one bare turn per period. And if the first text has not drained from the queue yet, the withdrawal leaves two queued wakes that cannot be cancelled.
- **Clearing the grant by hand** in `primary-sessions/<seat>.json`: the running receiver owns that file, and a write outside its lock races the redemption path.
- **Evicting an urgent grant whenever an ordinary record arrives**: that would drop urgent interrupts under ordinary traffic, the reverse of #1742.

## Related

#1742 / PR #1759 (the urgent slot) · #1741 · #503 · #1373 (its detector counts failed dispatches; here nothing fails, the ordinary digest is only deferred) · #1803 / #1805

Until this lands: an unredeemed attempt clears only when the seat's Codex app restarts or the seat starts a new main thread; a review request to an affected seat needs a manual nudge.

Live latest-open + closed sweep: newest 20 open Brain issues (156 open, red band), plus open and closed searches (`urgent wake`, `urgentInflight`, `unclaimed grant`, `primary-session wake`, `receipt-unproven`) at 2026-11-17T00:1xZ. #1742 is closed (it introduced the slot); #1741 is open and covers the other direction. Nothing owns this hold, so it is filed as a leaf despite the red band.
MC sweep: "urgent wake stays unresolved and ordinary wakes for the seat stay pending behind it" found no prior decision.
Own-assignment sweep: #503 (mine, the health projection) and #1373 (unassigned, failure-count detector); neither owns this hold.
A2A sweep: up to 23:58Z, Euclid's incident DM `bc1cecef`; no claim.

Origin Session ID: e9f42310-21e7-4c15-aada-079042a61067
Retrieval Hint: "unredeemed wake grant holds ordinary wakes", "wake turn completed without hook acknowledgement", "urgentInflight unknown same Codex session"

— Vega (Claude Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-11-17T00:03:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-17T00:03:31Z @neo-opus-vega added the `bug` label
- 2026-11-17T00:03:31Z @neo-opus-vega added the `ai` label
- 2026-11-17T03:57:56Z @neo-opus-vega cross-referenced by PR #1825
- 2026-11-17T03:58:11Z @neo-opus-vega cross-referenced by #503
- 2026-11-17T06:19:50Z @tobiu referenced in commit `92f0e73` - "feat(wake): a wake turn that ends without the hook's acknowledgement settles its slot (#1824) (#1825)

An attempt the recipient hook never redeemed held its slot until the
seat's session changed, and every ordinary wake waited behind an
unredeemed urgent one.

readCodexWakeReceipt now reports turn-unacknowledged when the exact wake
text started its queue turn and that turn completed or aborted without
any hook acknowledgement. Such an attempt is final: the interrupt
landed, the digest did not, and the prompt is never submitted again.
With no claim for the nonce, the fleet reader asks for that state, and
deliverPrimarySessionWake settles the slot as delivered with
acknowledged: false; the flag travels through the spool result to the
receiver record. Nothing is withdrawn or re-sent: in a session whose
hook is broken, a re-send fails the same way. An attempt whose claim
was admitted never settles like this, and a hook that runs after the
settlement finds no grant."
- 2026-11-17T06:19:51Z @tobiu closed this issue

