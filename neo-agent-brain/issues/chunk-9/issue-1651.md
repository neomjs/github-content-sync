---
id: 1651
title: Escalate a silent holder only after a receiver-confirmed delivery
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T16:21:49Z'
updatedAt: '2026-11-13T05:07:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1651'
author: neo-opus-grace
commentsCount: 3
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
closedAt: '2026-11-13T05:07:06Z'
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

## Intake (2026-11-12, after #1652 merged)

- **Readiness:** the only blocker, #807, closed with #1652 (`dc3082cc`). Parent #759 has an independent epic review: Euclid, [5957036101](https://github.com/neomjs/neo-agent-brain/issues/759#issuecomment-5957036101).
- **Drift since filing (Brain `857fe415`):**
  - #1652 delivered the exclude set and the holder-keyed escalation slots this leaf builds on.
  - #1657 makes the rotation pass over a plan-exhausted seat until its reset (`planOpenWorkWakes({exhaustedUntil})`, then `rotationRoute`). `holder-silent` picks through the same `rotate`, so it inherits that unchanged.
  - #1654 and #1656 touch no path named here.
- **Premise re-verified at `857fe415`:**
  - The round still discards the receipt: `try { await admitWake(observationOf(wake, failedChecks)) } catch {}` in `wireFleetOpenWorkWakes.mjs`.
  - `CoalescingEngineService#_getSourceEventId` falls back to `payload.messageId`.
  - Receiver records key on `sourceEventIds` (`receiverState.mjs#getWakeRecordKey`).
- **Prescription checked:** `ai/services/memory-core/wakeDeliveryProjection.mjs` owns the concern. `projectMessageDelivery` sits beside `projectWakeDelivery` (per subscription) and `projectIdentityWakeReachability` (per identity).
- **Read owner, sharpened:** `readWakeDelivery({recordsDir})` (`wakeDeliveryReader.mjs`) is already consumed by `WakeSubscriptionService`, whose `whoIsOnline` publishes `agents[].wake`, and by `HealthService`.
  - The per-message read becomes a `WakeSubscriptionService` method, so the records keep their one reader.
  - The MCP tool sits beside `admit_open_work_wake` only for its operator admission and extended tier.
- **Carried in:** Euclid's non-blocking note on #1652. `admitOpenWorkWake`'s JSDoc argument list (`openWorkWakeIntake.mjs`) omits `holder`, and this leaf extends that intake, so it fixes the list.
- **Classification:** valid as written, sharpened above. ROI is positive: this leaf completes #807's third escalation.

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
### @neo-opus-grace - 2026-11-12T21:24:28Z

**Handover (Grace, session sunset, solo-refresh, 2026-11-12 21:3xZ).** The owner is unchanged (@neo-opus-grace); the next session of this seat continues.

**State:** the intake is recorded in the body (see "Intake" above). Branch `grace/1651-holder-silent` exists in a seat worktree at `857fe415`, with no edits and nothing pushed.

**Implementation order.** Each step is red-first:
1. `projectMessageDelivery(records, messageIds)` in `wakeDeliveryProjection.mjs`. It is pure. An id found in a record's `sourceEventIds` takes that record's terminal state and time; an id in no record reads `undelivered`; unreadable records read `unknown`.
2. The read path:
   - a `WakeSubscriptionService` method that reads the records through `readWakeDelivery` and projects them;
   - the MCP tool beside `admit_open_work_wake` (`openapi.yaml`, extended tier, operator-admitted);
   - the in-process path for host mode (`devFleetServer.mjs`).
3. The round keeps each intake receipt's `messageId` on its admitted ledger entry, and `validateLedger` accepts it as optional (AC-1).
4. `holder-silent`:
   - the kind in `openWorkWakeMessage.mjs`: fixed text, `holder` required and never the recipient;
   - the planner escalates once a holder slot is still open 4 h after a `delivered` verdict;
   - specs that discriminate failed, undelivered and delivered (AC-2, AC-3).

   The same step adds `holder` to `admitOpenWorkWake`'s JSDoc argument list.
5. The PR, with a non-Claude reviewer. After merge: the plane cut first, then the FM update. AC-4 is due a week after the switch-on.

**Pickup:** start with `context-recovery`, read this body's Intake section, then begin at step 1 in the worktree.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 41210ffd-ab0e-42b0-b871-ad319bef1fed

- 2026-11-13T02:04:39Z @neo-opus-grace cross-referenced by PR #1659
- 2026-11-13T02:11:54Z @neo-opus-grace cross-referenced by #1642
### @neo-opus-grace - 2026-11-13T02:53:01Z

**Handover (Grace, session sunset, solo-refresh, 2026-11-13 02:3xZ).** The owner is unchanged; the next session of this seat continues.

**State:** PR #1659 @ `74dbf37f` is green and MERGEABLE, with Euclid requested as the GPT-family reviewer. The first CI attempt's red (`daemon.spec.mjs:1256`, a 10 s log poll) was falsified by a full re-run of the same head; [CI note](https://github.com/neomjs/neo-agent-brain/pull/1659#issuecomment-6841883857). The worktree `neo-agent-brain-1651` holds the branch, is pushed, and has its config overlays hydrated.

**Pickup:** answer Euclid's review per `review-response-protocol.md`: an author-response comment with the exact head, then the re-request at green. After merge, cut the plane first, then update the FM; in the other order the read fails, every verdict reads `unknown`, and nothing escalates. AC-4 is due a week after switch-on, recorded on #759. The live join-key witness is on [#759](https://github.com/neomjs/neo-agent-brain/issues/759#issuecomment-6841703543).

🖖 Grace (Claude Opus 5.5, Claude Code) · session d165694b-86ad-47cb-8342-4282d81ba3e3

- 2026-11-13T04:40:01Z @neo-opus-grace referenced in commit `4539a0f` - "fix(fleet): read every eligible holder receipt each round in plane-sized batches (#1651)

The round asked the plane about the oldest hundred admitted holder wakes only, so a failed or
already-escalated wake that stayed open held its place in that prefix, and a delivered wake behind
it was never read. It now reads every eligible wake each round, at most a hundred ids per call, and
a call that fails leaves only its own wakes unknown."
- 2026-11-13T05:07:06Z @tobiu referenced in commit `588362e` - "feat(fleet): escalate a silent holder only after a receiver-confirmed delivery (#1651) (#1659)

* feat(fleet): escalate a silent holder only after a receiver-confirmed delivery (#1651)

The round keeps each intake receipt's message id on its admitted slot and
reads the slot's delivery back through read_open_work_wake_delivery, an
operator-admitted plane read over the wake receiver's records. A holder slot
still open four hours after a delivered wake wakes one rotation seat as
holder-silent, once per head, never the holder; a sent escalation stays held
through a round whose read fails.

* fix(fleet): read every eligible holder receipt each round in plane-sized batches (#1651)

The round asked the plane about the oldest hundred admitted holder wakes only, so a failed or
already-escalated wake that stayed open held its place in that prefix, and a delivered wake behind
it was never read. It now reads every eligible wake each round, at most a hundred ids per call, and
a call that fails leaves only its own wakes unknown."
- 2026-11-13T05:07:07Z @tobiu closed this issue

