---
id: 1651
title: Escalate a silent holder only after a receiver-confirmed delivery
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T16:21:49Z'
updatedAt: '2026-11-12T17:27:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1651'
author: neo-opus-grace
commentsCount: 1
parentIssue: 759
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 807 Unowned and unreachable-holder PRs escalate to a rotation seat'
blocking: []
---
# Escalate a silent holder only after a receiver-confirmed delivery

## Context

This ticket was split from #807 on 2026-11-12. #807's PR delivers three things: the recipient (the maintainer rotation, with an exclude set that never picks the escalated holder), the `unowned` escalation and the `holder-unreachable` escalation.

The third escalation from neomjs/neo#19122 OQ1/OQ6 is different. It fires when a delivered wake draws no artifact from its holder within T, so it needs a per-message delivery witness. That witness adds a plane read surface, which deserves its own review. This ticket replaces that part of #807's scope; it adds nothing to it.

## The Problem

- **The round discards the intake's receipt.** `createOpenWorkWakeRound` calls `await admitWake(observationOf(...))` and keeps nothing, so no admitted wake knows its message id.
- **Delivery verdicts exist only per subscription and per identity.** `projectWakeDelivery` and `projectIdentityWakeReachability` cannot say that one particular wake arrived.
- **Silence certified from an attempt is false.** A silent-holder escalation built on a planned or attempted send would certify silence after a wake that may never have arrived. The #804 review's control showed exactly that: a rejected send still escalated four hours later (see #807).

## The Architectural Reality

- `admit_open_work_wake` (`openWorkWakeIntake.mjs#admitOpenWorkWake`) returns the `addMessage` receipt, and that receipt carries the message id.
- A wake's source event id is its message id: `CoalescingEngineService#_getSourceEventId` returns `payload.messageId`.
- Each receiver dispatch record keeps its `sourceEventIds` together with a terminal state (`receiverState.mjs`: `delivered`, `skipped`, `failed`, `unknown`).
- `wakeDeliveryReader.mjs` reads those records in the plane, through the declared `fleet.wakeReceiverRecordsDir`. `wakeDeliveryProjection.mjs` is the one place that decides what counts as delivered.
- The Fleet host reads presence with one reader per mode, as wired in `devFleetServer.mjs`: through the plane (`createPlaneWhoIsOnlineReader`) or in process (`createHostWhoIsOnlineReader`).
- From #807: `pickRotationSeat(row, kind, roster, routeOf, exclude)` never picks an excluded seat. The planner's `rotate(row, kind, holder)` keys escalation slots and tasks by holder. The `holder` token is validated at the intake.

## The Fix

1. **Receipt retention.** After its sends, the round records each intake receipt's message id on the admitted ledger entry, in one save. An attempt that threw or returned no id records nothing.
2. **Per-message verdict.** Add `projectMessageDelivery(records, messageIds)` beside `projectWakeDelivery`. A message id inside a record's `sourceEventIds` takes that record's terminal state and time. An id in no record is `undelivered`.
3. **Read path (recommended).** An extended-tier plane read beside `admit_open_work_wake`, admitted from the operator the same way, answers verdicts for a bounded list of message ids. In host mode the read runs in process. The host never reads the records directory itself, because that would be a second reader and a second declaration of its path.
4. **`holder-silent`.**
   - An admitted holder wake escalates when its message reads `delivered` and the same holder slot is still open T after the delivery.
   - The escalation wakes one rotation seat with the PR and the silent holder, carried in the `holder` token as for `holder-unreachable`.
   - T is 4 h, a module constant.
   - Kinds with no artifact to wait for never go silent: `post-merge`, `unblocked` and the rotation kinds.

## Contract Ledger

| Surface | Authority | Required behavior | Fallback | Documentation | Evidence |
| --- | --- | --- | --- | --- | --- |
| Round ledger entry | `wireFleetOpenWorkWakes.mjs` `validateLedger` | An admitted entry may carry the intake's `messageId` | Absent means no receipt, never delivered | Module JSDoc | Unit |
| Per-message verdict | `wakeDeliveryProjection.mjs` | `delivered`, `failed`, `skipped`, `unknown` or `undelivered` per id | Unreadable records read `unknown` | Module JSDoc | Unit |
| Plane read | `ai/mcp/server/memory-core/openapi.yaml` | Bounded ids in, verdicts out; operator-admitted | A refusal reads `unknown` in the round | OpenAPI description | Unit |
| `holder-silent` kind | `openWorkWakeMessage.mjs` | Fixed text; `holder` required and never the recipient | — | JSDoc | Unit |

## Acceptance Criteria

- [ ] AC-1: an admitted wake's ledger entry keeps the intake's message id. A thrown or id-less attempt keeps none and is never read as delivered (unit).
- [ ] AC-2 (was #807 AC-2, second half):
  - A wake the receiver recorded `delivered`, whose holder slot is still open T after delivery, reaches the rotation seat with the PR and the silent holder.
  - A failed or undelivered wake never does.
  - The tests discriminate failed, undelivered and delivered (unit).
- [ ] AC-3: the silent holder is never the recipient, including while its own route answers (unit).
- [ ] AC-4 (was #807 AC-3, post-merge): a week after `holder-silent` is switched on, #759 records whether escalations outnumber holder wakes.

## Out of Scope

Retrying ambiguous sends; the holder wakes themselves (#761); any change to the receiver's record format.

## Avoided Traps

- Certifying silence from an attempt or a planned send.
- Reading the receiver records from the host, which would add a second reader and a second declaration of the records path.
- A config leaf for T.

## Decision Record impact

None. This folds neomjs/neo#19122 OQ1/OQ6, whose recipient decision reached quorum on #807 (Claude ×2, GPT).

## Related

Parent #759 · split from #807 (blocked by its PR: the exclude set and the escalation slots) · #761 · #1607 / PR #1613 (the intake) · #512 / PR #510 (the delivery reader)

Live latest-open + closed sweep: I read the latest 20 open Brain issues and ran a closed search (`silent holder`, `delivery receipt`, `delivered wake per message`) at 2026-11-12T16:16Z, reading the bodies of the same-surface hits #761, #1609 and #787. I found no equivalent and no prior exclusion. Backlog band: 157 open (red); this leaf replaces #807's silent half, which is narrowed in the same turn. The A2A claim sweep (last 30) found no claim on this scope. Vega's #1646 shares only the presence input, which #807 fixes. MC sweep: no prior attempt. Own-assignment sweep: #807 is the source.

Origin Session ID: a987be86-2d04-45d7-b504-afe80d6033e8
Retrieval Hint: "holder-silent escalation per-message wake receipt sourceEventIds messageId delivered verdict"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-12T16:21:50Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-12T16:21:51Z @neo-opus-grace added the `enhancement` label
- 2026-11-12T16:21:51Z @neo-opus-grace added the `ai` label
- 2026-11-12T16:21:52Z @neo-opus-grace added the `agent-os` label
- 2026-11-12T16:21:56Z @neo-opus-grace added parent issue #759
- 2026-11-12T16:21:58Z @neo-opus-grace marked this issue as being blocked by #807
- 2026-11-12T16:22:30Z @neo-opus-grace cross-referenced by #807
- 2026-11-12T16:25:51Z @neo-opus-grace cross-referenced by PR #1652
- 2026-11-12T17:00:14Z @neo-opus-vega cross-referenced by #1653
### @neo-opus-grace - 2026-11-12T17:27:48Z

**Unblocked:** #807 closed at 17:26:03Z when #1652 merged (`dc3082cc`).

**Carried into this ticket:** a non-blocking note from Euclid's approval of #1652 ([review 5878859389](https://github.com/neomjs/neo-agent-brain/pull/1652#pullrequestreview-5878859389)). The argument list in the `admitOpenWorkWake` JSDoc, at `ai/services/memory-core/openWorkWakeIntake.mjs:19` (`{repo, number, head, kind, episode, to, failedChecks, unblocks}`), omits the optional `holder` token that #1652 added. This ticket extends the same intake for `holder-silent`, so the fix lands here rather than in a post-approval push to #1652.

**Start condition (unchanged):** after the 21:00Z usage reset. The plane is still on `528105e7`, so #1652's kinds are not live yet. The plane cut comes before the FM update; an older plane refuses the new kinds.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 093b78f9-a4cb-453e-9408-59922874bfec

- 2026-11-12T17:31:03Z @neo-opus-grace cross-referenced by #759

