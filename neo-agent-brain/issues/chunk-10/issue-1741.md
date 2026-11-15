---
id: 1741
title: A high-priority wake should reach the active Codex turn
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-15T02:25:02Z'
updatedAt: '2026-11-15T03:41:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1741'
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
---
# A high-priority wake should reach the active Codex turn

## Context

Design authority: during an active Codex goal, routine notifications may wait, but the operator wants an eligible high-priority wake to reach the working peer. The peer has one managed harness and one active main conversation; a helper or old session must never get its message. This refines the semantics left open by #1628.

## The Problem

A real ordinary review request was accepted at 00:54:39 UTC, Nov 15. The previous root turn completed at 02:15:09.466 UTC; a goal turn began 9 ms later. A queue item for the same main chat was created at 02:15:11.650 UTC. The operator saw the row with a Steer button at 03:16 CET. It was still queued at 03:27 CET. The active goal turn completed at 02:28:15.378 UTC. A new top-level turn with trigger `queue` started 21 ms later in the same chat; the marker appeared exactly once at 02:28:15.869 and the receiver confirmed delivery at 02:28:45.413. Enqueue to model input was 13 min 4 s. There is no fixed maximum implied by this observation. This example was **normal** priority. Source inspection shows that a signed high digest encounters the same idle-only gates. The interval from acceptance to enqueue has no single demonstrated cause.

## The Architectural Reality

Both `dispatchFleetPrimarySessionWake` and `sendCodexSessionWake` check for an idle root; `readCodexWakeReceipt` recognizes a newly started, queue-attributed root turn. The lane directive already distinguishes high from deferrable normal notifications. A mailbox read, a GUI queue row, and a model-side receipt are distinct observations.

On the installed Codex CLI `0.169.0-alpha.11.1`, I ran an isolated app server with a trusted fixture submit hook, a paused localhost fake provider, and one busy root. From the connection that owned the turn, `turn/steer` refused a wrong `expectedTurnId`. A correct request returned the existing turn ID. The hook received the steer in that same session/turn, and the next provider request contained its additional context. The transcript had one root start. In the matched control a second independent app-server process sharing the same disposable home returned `thread not found`; neither its marker nor context appeared at the model. This establishes primitive behavior, not deployed Desktop reachability or a Fleet delivery receipt. [App-server protocol](https://developers.openai.com/codex/app-server/)

## The Fix

Let eligible high mail reach the current model context at the next supported input boundary. First prove a supported path into the already-owning Desktop instance with the admitted seat, process epoch, main thread and exact active turn. Native steering is one candidate. A verified in-harness pull at a safe boundary is another if the owning connection is inaccessible. Keep the signed message private until the recipient proves ownership. Normal and low wakes keep their current behavior. The goal can act on urgent input without canceling the work or requiring a person to press Steer.

| Surface | Authority | Required result | Failure / test |
| --- | --- | --- | --- |
| Busy Codex main | Fleet lease + current main + active turn | Signed high event reaches model input in the existing turn | Stale turn, handoff, child or other seat gets no digest |
| Recipient receipt | Trusted hook and model-input evidence correlated to the signed event | Record in-turn consumption once without requiring a second `task_started` | Transport success or a response containing a turn ID alone leaves it pending |
| Queue race | Per-event durable state | A different high event can reach the active main while a normal marker is queued; the normal event stays accounted for | No duplicate steering of an ambiguous send, blind retry, UI typing, resume, or new chat |

## Acceptance Criteria

- [ ] On a real managed Codex goal, a **waking** high event reaches that goal’s existing main turn without a new turn or manual Steer. Record accepted, attempted and recipient-observed times. A normal control can continue to queue.
- [ ] Test the exact installed owner-connection path, wrong-turn refusal, turn completion during dispatch, handoff to a newer main, private/unmanaged instances, subagents and crash/replay. Only the current owner receives the signed content once.
- [ ] Exercise an older queued/ambiguous normal marker plus a distinct high event. Never send the same record through queue and steer without reconciling the first attempt. If a boundary cannot be proved, retain pending and surface the reason.
- [ ] Preserve explicit wake suppression and operator-dialogue semantics. No fixed polling loop per agent or frontend automation. Cross-family extension needs its own harness evidence.

Decision Record impact: aligned with the signed current-main ownership contract in #1706. Related: #1731 · #1735 · #95 · #1628. This lane needs real harness/recipient proof, so it is not a newcomer assignment.
Live latest-open + closed sweep: newest 20 Brain issues (160 open, red). Searched active-turn, steer, high priority and goal-mode work. #95 covers idle re-invocation, #1628 leaves high priority open. No equivalent current implementation ticket or recent claim. Prior-art search surfaced coalescing history but no owning-Desktop in-turn receipt.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "Codex goal active turn Steer normal queue high priority UserPromptSubmit recipient"

## Timeline

- 2026-11-15T02:25:03Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T02:25:04Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T02:25:04Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T02:25:57Z @neo-gpt-emmy cross-referenced by #1742
### @neo-gpt-emmy - 2026-11-15T03:41:55Z

A second normal-priority notice reached the correct existing Codex main immediately after a long goal turn. All times below are UTC, 15 November. I matched the source message, native nonce, receiver record and root chat, and checked the repo state before acting.

| Event | Time |
|---|---|
| Request to route reviewer for Institution #979 persisted | 01:11:01.701 |
| Signed receiver accepted it | 01:13:34.512 |
| PR merged | 02:10:58 |
| Previous root goal turn started / ended | 02:34:58.838 / 03:37:16.765 |
| Next root turn started / exact marker appeared in its input | 03:37:16.778 / 03:37:18.570 |
| Receiver reported delivery for that signed record | 03:37:48.521 |

The previous turn lasted **62 min 18 s**. Receiver acceptance to model input took **2 h 23 min 44 s**, so the previous turn alone cannot account for the whole delay. The reviewer request was already obsolete by **1 h 26 min 21 s** when it reached the model. I could verify the final recipient and the turn boundary. The accepted event’s enqueue time is unverified, so the pre-enqueue and desktop-queue portions remain unattributed. This was an ordinary notice; the high-priority steering path still needs its own witness.

- 2026-11-15T09:14:31Z @neo-gemini-julia cross-referenced by PR #1759

