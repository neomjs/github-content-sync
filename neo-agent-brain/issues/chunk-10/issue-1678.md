---
id: 1678
title: 'A failed open-work wake send is never retried, so its holder never hears'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-13T09:25:30Z'
updatedAt: '2026-11-13T15:15:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1678'
author: neo-opus-grace
commentsCount: 0
parentIssue: 759
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-13T15:15:25Z'
---
# A failed open-work wake send is never retried, so its holder never hears

## Context

The Fleet host's open-work wake round persists a holder's slot as `admitted` before it sends. An attempt that throws or times out is logged as *"open-work wake attempt failed or is ambiguous; admission retained"* and is never sent again. On the installed host (`857fe415`), the harness log shows five such sends between 2026-11-11T20:17Z and 2026-11-13T09:22Z:

| At (UTC) | PR | Kind |
|---|---|---|
| 11-12 17:44:09 | neomjs/neo#20827 | route-review |
| 11-12 17:45:17 | neomjs/neo#20827 | review |
| 11-13 04:26:07 | neomjs/neo-agent-institution#938 | review |
| 11-13 05:55:07 | neomjs/neo-agent-institution#950 | review |
| 11-13 09:10:14 | neomjs/neo-pro#228 | holder-unreachable |

The #950 case end to end:
- **The ledger:** the host's `open-work-wakes.json` holds the slot `[…#950, a8d22f5c, @neo-gpt-sophie, review]` as `admitted`.
- **The message:** none exists; Institution #490's trace receipt 6848805053 (observation 5) reads no `[open-work:review]` to the reviewer between 04:56Z and 09:04Z.
- **The cost:** the reviewer's card held the row as `review-due` for 3.5 hours, until a manual ping.
- **The plane:** it was slow around that time. Roster wake reads answered after 4.4–9.8 s against a 4 s bound (log, 05:54–06:03Z).

## The Problem

The planner says "admitted" means reserved, never delivered. The round persists the ledger first and then calls `admitWake`; on a throw it logs and keeps the slot admitted. The next pulse finds the slot admitted and does nothing, so that episode never wakes its holder.

The host cannot simply retry. The plane's `admitOpenWorkWake` writes a new `@system` message on every call. After an ambiguous failure, such as a timeout after the plane wrote, a retry would wake the holder twice. That is why #761 placed the dedupe key in the producer, and #807 left "retrying ambiguous sends" out of scope. Nothing has owned the lost wake since.

## The Architectural Reality

- **Planner:** `ai/services/fleet/openWorkWakes.mjs#planOpenWorkWakes`. A slot is `[repo, number, head, seat, kind]`, its status `baseline | pending | admitted`, and a wake's `key` adds the episode. The plane's task id already carries the same identity: `open-work:[repo, number, head, to, kind, episode(, holder)]`.
- **Round:** `ai/services/fleet/wireFleetOpenWorkWakes.mjs`. It persists before sending, and three paths leave an admitted slot unsent: disabled mid-loop, an expired wake (`freshWake`), and a throwing `admitWake`.
- **Plane intake:** `ai/services/memory-core/openWorkWakeIntake.mjs#admitOpenWorkWake` calls `MailboxService.addMessage` as `@system` with no lookup by the wake's task.
- **Delivery read:** on `dev`, the round keeps the intake receipt's message id on an admitted slot (#1659), but the installed host predates it. A failed attempt has no receipt, so it is never certified silent either.

## The Fix

- **Plane:** make admission idempotent on the wake's task. A second admission of the same task answers the first message's receipt instead of writing again. The record may live in the plane's process; a restart between a lost answer and its retry is the accepted residual.
- **Host:** the round can then keep a failed attempt retryable. A throwing `admitWake` leaves the slot retryable, so the next pulse sends it again. An expired wake stays expired, and a refused observation is never retried.
- **Mechanism and bounds:** the owner picks the plane-side lookup. Retries are bounded per slot, and nothing here adds a new config.

## Acceptance Criteria

- [ ] AC-1: a second `admit_open_work_wake` of the same wake writes no second message and answers the first receipt (plane intake, unit).
- [ ] AC-2: a send that throws is attempted again on the next pulse and wakes the holder exactly once, including when the first attempt timed out after the plane wrote (round + intake, unit).
- [ ] AC-3: retries are bounded per slot, and a refused observation is not retried (unit).
- [ ] AC-4: post-merge, after the plane cut and the Fleet install, the host log shows no lost wake for one day, or each failed attempt followed by an admitted retry.

## Out of Scope

- Receiver-side delivery: a subscription that reports deliverable while dispatch fails (#503), or a seat's GUI paste failing.
- The plane's latency.
- The wake digest (#762).
- The expired and disabled paths, which no measured loss involves.

## Avoided Traps

- **Retrying on the host without plane idempotency:** an ambiguous timeout becomes a duplicate wake, which is why #761 placed dedupe in the producer.
- **Re-arming the slot as `pending` alone:** the same duplicate under the same ambiguity.

## Related

#759 (epic) · #761 · #807 · #1659 · #503 · neomjs/neo-agent-institution#490 (trace observation 5) · neomjs/neo-agent-institution#950

Ownership: self-assigned, under my #759 lane.

Body corrected 2026-11-13: the message-id AC and the "records none" line described the installed `857fe415`; on `dev` the round keeps the receipt since #1659.

Live latest-open + closed sweep:
- **Open/closed search:** checked the latest 20 open issues and a closed search ("open-work wake admission retained retry", "wake send failed ambiguous duplicate") at 2026-11-13T09:28Z. I read the retry lines of #761 and #807: #761 places dedupe in the producer, and #807 excludes retrying ambiguous sends. No equivalent ticket or prior exclusion of the defect itself.
- **Backlog band:** 159 open (red). Folded as a leaf under the owning epic #759.
- **Memory Core sweep:** earlier missed-wake records are receiver-side (webhook, deadlock), not a lost send.
- **Own-assignment sweep:** #1642, #1630 and #762 sit in the same lane; none covers a failed send.
- **A2A claim sweep:** no overlapping claim in the last 60 minutes.
- **Structure map:** the three modules sit in `ai/services/fleet` and `ai/services/memory-core`, with no placement change.

Origin Session ID: 66036243-460b-4235-af05-cff4124faafb
Retrieval Hint: "open-work wake attempt failed or is ambiguous admission retained never retried holder never woken"

## Timeline

- 2026-11-13T09:25:30Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-13T09:25:32Z @neo-opus-grace added the `bug` label
- 2026-11-13T09:25:32Z @neo-opus-grace added the `ai` label
- 2026-11-13T09:25:32Z @neo-opus-grace added the `agent-os` label
- 2026-11-13T09:25:40Z @neo-opus-grace added parent issue #759
- 2026-11-13T09:41:32Z @neo-opus-grace cross-referenced by PR #1680
- 2026-11-13T13:12:34Z @neo-gpt-sophie cross-referenced by #842
- 2026-11-13T13:23:25Z @neo-opus-ada cross-referenced by #1681
- 2026-11-13T14:31:31Z @neo-opus-grace referenced in commit `089ca12` - "fix(fleet): a failed open-work wake send is retried, and the plane answers a repeat with its first receipt (#1678)"
- 2026-11-13T15:15:24Z @tobiu referenced in commit `9b33cc8` - "fix(fleet): a failed open-work wake send is retried, and the plane answers a repeat with its first receipt (#1678) (#1680)"
- 2026-11-13T15:15:26Z @tobiu closed this issue

