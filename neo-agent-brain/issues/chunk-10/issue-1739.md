---
id: 1739
title: 'Amendments, reconnects and a deferred count strand a sunset rotation'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-15T02:01:32Z'
updatedAt: '2026-11-15T14:21:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1739'
author: neo-opus-ada
commentsCount: 9
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
closedAt: '2026-11-15T05:07:56Z'
---
# Amendments, reconnects and a deferred count strand a sunset rotation

## Context

Design authority: the operator's ruling on 2026-11-14 (#1722): *"a sunset should be terminal. session close and start a fresh session instantly."* A second live session of one seat *"violates 'never a copy'"*. On 2026-11-15, the operator watched the specimens below and said: *"if FM would rotate while grace is already running a session, this would be bad."*

Three live specimens on plane `2cb8e911`, 2026-11-15. The first rotation after the cut (Ada's own) passed: #1722 comment 6887551864.

**Grace.** Receipt `46a3a54f`, sent 01:41:58Z. Times come from her A2A `6f525ede`.

| time (Z) | event |
|---|---|
| 01:42:28, 01:42:37 | two background subagents finish, and each re-invokes the sunsetted session |
| 01:42:44.95 | the plane requests the rotation |
| 01:43:03.2 | typed; the fresh session starts |
| 01:43:49 | the fresh session's marker `d1c350a2` replies to `46a3a54f` |
| 01:44:23 | the old session sends an addendum `ac5bc324`: handover tag, no focus items |
| 01:45:03.1 | the plane reads `ac5bc324` as a new sunset and refuses it (`minimum-interval`) |
| 01:45:59 | woken by a third subagent, the old session sends v10.1 `e2a6f73c` with the same tag |
| 01:46:35 | the old session posts a GitHub comment on neo `#20983` |
| 01:47:07.9 | `e2a6f73c` is refused (`minimum-interval`) |

`46a3a54f` never completes, although its marker landed 34 s before the addendum.

**Mnemosyne.** Receipt `16d9a912`, sent 01:50:08.8Z, 5 focus items placed. Run read-only at 01:56:55Z, the plane's own `readSunsetReceipt` and `decideRotation` return `wait: live-turn` until 02:49:21.9Z:
- Her closing `add_memory` (01:50:35.7Z) is saved under Memory Core session `8a74258a`.
- Her last diary before the receipt (00:52:42Z) is under `0bb3dc6b`, so the plane takes `0bb3dc6b` as the writer.
- It refuses the close as one "first seen after the receipt" and waits for her one open interval, a post-tool-use interval open since 01:10:47Z, to expire. The rotation runs about 59 minutes late.
- The session change inside one turn is a bridge reconnect, measured on Ada's seat in comment 6887888097.

**Vega.** Receipt `cb097055`, sent 02:23:24.5Z.
- Her `add_memory` closed the turn under her writer session at 02:23:56.9Z, and the plane attributed it.
- The receipt was never counted. The summary lane logged `Deferring session summarization; heavy maintenance task tenant repo sync (cloud) is active` at 02:19:30Z, so `decideRotation` waited on `receipt-count-pending`.
- At 02:26:30.5Z the operator, seeing no fresh session, typed into her old session. The plane refused at 02:26:38Z (`prompted-after-receipt`). That guard was right; the silent wait before it was the failure.

Instruments: the orchestrator's `session-rotation:` and summary-lane lines; the receiver records `session-rotate:MESSAGE:…`; read-only `better-sqlite3` reads in the orchestrator container, running the plane's own read functions plus metadata only.

## The Problem

`readSunsetReceipt` infers three answers, and each inference broke.

1. **Which message is the receipt.** It takes the seat's newest self-message that is not a successor marker, and a marker counts only when it replies to that one. So a handover sent after the request does two things:
   - **It strands the open rotation.** No marker can complete `46a3a54f` now, and the plane never revisits it, so it never turns `incomplete` either.
   - **It is decided as a new sunset.** Only the 30-minute `minIntervalMs` refused both amendments. The same handover sent later is requested once its turn closes, and it types a fresh session into the seat's app although the seat's current session never sunset.
2. **Which session wrote it.** The receipt names no sender session (defect-note `e3ad54e4`), so `oldSessionId` is the session of the last memory at or before the receipt. A writer whose Memory Core session changed after its last diary cannot close its own receipt, and the rotation waits up to an hour for its intervals to expire.
3. **Whether it is counted.** The rotation waits until the summary lane stamps `focusItemsReceipt`, and heavy maintenance defers that lane. The count sits in the receipt's own subject (`focus items: N, N placed`), so waiting on a heavy run for it holds every sunset behind maintenance. #1710 recorded this coupling as Out of Scope.

Emmy's Round 1 on PR #1740 adds the generation question.
- **A writer is not a generation.** A handover sent after a rotation's successor marker may come from the old session.
- **No Memory Core id proves which generation wrote it.** A fresh session can inherit the old id (comment 6887683226), and a reconnect in the old session mints a new one (her witness on PR #1740).

## The Architectural Reality

- `ai/daemons/wake/sunsetReceipt.mjs` `readSunsetReceipt`:
  - selection: the newest run of handovers, `marker`;
  - writer inference: `oldSessionId`, from the last `AGENT_MEMORY` at or before `sentAt`;
  - close: `writerSaves`, `savedBy`, `closedAt`;
  - other-session guard: `before`, `otherSaveAt`.
- `ai/daemons/wake/sessionRotation.mjs` `decideRotation`: a terminal receipt yields `none`. Once delivered, only a marker or the window settles a rotation. An unstamped count waits (`receipt-count-pending`).
- `ai/daemons/orchestrator/scheduling/summary.mjs`: `parseFocusItemsReceipt`, and the `sunset-handover` trigger, whose `onSuccess` stamps the count only after the heavy summarization run.
- Three consumers read the same answer: `runSessionRotation` (`ai/daemons/orchestrator/scheduling/sessionRotation.mjs`), the wake hold (`WakeSubscriptionService.holdsWakesForRotation` → `holdsWakes` → `isReceiptOpen`), and `ai/scripts/lifecycle/checkSunsetted.mjs`. A fix in the read keeps all three on one answer.
- `MailboxService.addMessage` keeps a caller's `originSessionId` only as an `ORIGINATES_IN` edge (`optionalEdges`), never as a message property, and only when the caller passes one. `add_memory` already defaults its `sessionId` to `SessionService.currentSessionId` (`MemoryService.mjs:664`). Turn-presence intervals carry no session (`TurnPresenceService`), so the read has no other source for the writer.
- Structure map: no new file; the owners are `ai/daemons/wake`, `ai/daemons/orchestrator/scheduling` and `ai/services/memory-core`. No AiConfig leaf changes; `windowMs` reaches `decideRotation` as it already does. Decision Record impact: none (a correction inside #1383's rotation contract).

## The Fix

1. **An amendment is not a new receipt.** A handover sent while an earlier receipt's rotation is open, and before that rotation's first successor marker, folds into that receipt; a marker replying to any message of the chain completes it. An open rotation is returned before any later receipt, so it always completes first.
2. **The receipt names its writer.** On a self-message tagged `sunset-protocol-handover`, `addMessage` defaults `originSessionId` to `SessionService.currentSessionId`, the source `add_memory` uses, and keeps it as a message property beside the existing edge. Without a predecessor (item 4), `readSunsetReceipt` takes the writer's session from that property when it is present, and falls back to today's inference for older receipts. The stamp attributes the writer's close; it never identifies the native holder. Other traffic is unchanged: no system or peer message gains a stamp or an edge.
3. **The receipt counts itself.** `readSunsetReceipt` takes the lane's stamp, else the subject's count via `parseFocusItemsReceipt` (moved into the pure `sunsetReceipt.mjs`), else `'missing'`. Both rotate, so `decideRotation` drops the pending-count wait. The summary lane still stamps and still publishes its missing-count defect note.
4. **After a predecessor's marker, only an attributed close releases a receipt.** A handover sent after a requested rotation's successor marker carries `predecessor` (`successorAt`).
   - There the seat's newest saver attributes the close, as before this ticket. The stamp is still recorded, but alone it never releases a live turn, since it may be the predecessor's late writer.
   - With no such close, `decideRotation` keeps the receipt pending: `wait: writer-close-pending`, logged, nothing sent. It stays pending until a close attributes it or its window passes.
   - No marker age decides it.

## Acceptance Criteria

- [ ] Unit (orchestrator spec), red on `dev` first, Grace's shape: receipt R is requested and delivered, a marker replies to R, then handovers arrive after the marker. R completes first. The later handovers are never requested, and once R settles the newest stays pending (`writer-close-pending`) with nothing sent.
- [ ] Unit: a marker replying to an amendment sent before any marker completes R.
- [ ] Unit: without a predecessor, the newest handover stays the receipt (today's behavior, including `minimum-interval`).
- [ ] Unit (orchestrator spec), red on `dev` first, Mnemosyne's shape: the receipt carries `originSessionId` S, the last diary before it is under another session, and the close is S's `add_memory`. The verdict at the next tick is `request`, not `wait: live-turn`.
- [ ] Unit: a receipt without `originSessionId` keeps today's inference.
- [ ] Unit (mailbox spec): a handover-tagged self-message sent without `originSessionId` carries the server's current session as a property; any other message gains none.
- [ ] Unit (orchestrator spec), red on `dev` first, Vega's shape: an unstamped receipt whose subject carries `focus items: 6, 6 placed` reads `{named: 6, placed: 6}` and is requested at the next tick. One without a count reads `'missing'` and is requested too. A stamp wins over the subject.
- [ ] Unit (orchestrator spec), red on the previous head, Emmy's overlap controls: the old writer's counted addendum while the successor's turn is live sends nothing, 38 minutes and 3 h 13 min after the marker. This holds with or without successor progress since, and when its closing save carries a reconnected Memory Core id. It sends nothing after the successor's turn expires either.
- [ ] Unit, red on the previous head, the inverse ordering: a successor's sunset sent after its marker but before the completion stamp waits while the predecessor completes. Then, with no close in view, it stays pending and sends nothing, whatever its stamp.
- [ ] Unit, red on the previous head, Emmy's positive control (the seat's own sunset 1 h 43 min after its marker on 2026-11-15): a successor's own sunset rotates once its own close lands, 1 h 43 min and 40 minutes after the marker alike, and stays pending before that close.
- [x] Post-merge (live): the next Claude sunset with an amendment, with a writer whose session changed mid-turn, or sent while heavy maintenance runs is requested within one lane tick of its `add_memory` and completed by its marker. Ada's 13:55Z sunset (6895923758): requested 62 s after its `add_memory` (60 s tick) while heavy maintenance ran and after two session-id changes, completed by its marker at the next tick.
- [x] Post-merge (live, owned here from neomjs/neo-agent-skills#395 AC-5): the next Claude sunset that ran background work shows no turn from its old session after the handover. Same sunset (6895923758).

## Out of Scope

- **The sunset precondition that no background task is in flight.** Grace's old session was re-invoked three times by its own subagents after its sunset. That fix belongs to the session-sunset skill: neomjs/neo-agent-skills#395, merged as PR #396.
- **Proving a generation (D#19401).** Two cases need a native holder or turn proof, such as a keyed hash of the hook payload's `session_id` on each interval. Until that lands:
  - A post-marker receipt whose writer's close comes under an id the newest saver does not hold (a reconnect inside the successor's sunset turn) stays pending until its window passes. Releasing it early is the binding's job.
  - A late writer that shares the successor's Memory Core id cannot be told from the successor.
- **The summary run's own place behind the heavy-task gate.** It still waits for maintenance; only the count stops waiting for it.
- **The FM's `next <time>` label, and its silence during waits (Institution).** Defect-note `e276b4ca`.
- **`minIntervalMs` and `windowMs` values.**

## Avoided Traps

- **Treating every handover that replies to an earlier one as an amendment.** A real next sunset may thread to its predecessor's handover and would then never rotate.
- **Telling amendments apart by content (focus items).** A complete second handover is still an amendment.
- **Trusting any close after the receipt.** The "first seen after the receipt" rule exists because another live session can write that close. The receipt's own session is what makes the close attributable.
- **Reading `originSessionId` as proof of a generation.** A fresh session can inherit the old id, and a reconnect in the old session mints a new one (Emmy's witness). The stamp attributes a close and nothing more.
- **Letting an attributed close clear another session's hold.** Emmy's Round 1: a close can be correctly attributed and still have no authority over a successor's live interval.
- **A time window as generation proof.** Three hours after a marker, the previous head had two failures (Emmy's Round 2):
  - It refused a successor's own sunset 1 h 43 min in, so that seat never rotated.
  - It rotated a counted late overlap.

  Age governs a receipt, not who owns a live interval.
- **Saving the diary before the receipt to dodge the close.** Without a close after the receipt, the plane waits on presence alone, up to an hour.
- **Raising `minIntervalMs`.** It narrows the hole without closing it, and it delays the loop guard.

## Related

#1722 (closed by PR #1730; live record 6887551864) · #1712 · #1710 · #1401 · #1383 · D#19401 · neomjs/neo-agent-skills#395

Live latest-open + closed sweep: latest 20 open plus a closed search ("sunset handover receipt rotation"; Out of Scope read on #1712 and #1710) at 2026-11-15T01:53Z, no equivalent. Backlog 159 open (red), and no open ticket owns rotation-receipt defects, so this one is filed. A2A: the latest 30 at 01:53Z and the latest 8 at 01:58Z, no claim on the receipt read. MC: two problem-noun queries, my own trail only. Own assignments: none on this surface.
Labels: not `good first issue` / `help wanted`. Claude seats' night rotation depends on it, and I own the rotation lane.

Retrieval Hint: "sunset receipt amended handover addendum shadows open rotation marker strands minimum-interval originSessionId writer session first seen after receipt live-turn wait focus items count summary lane deferred heavy maintenance receipt-count-pending predecessor successor writer-close-pending newest saver generation reconnect window"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T02:01:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-15T02:01:33Z @neo-opus-ada added the `bug` label
- 2026-11-15T02:01:34Z @neo-opus-ada added the `ai` label
- 2026-11-15T02:01:34Z @neo-opus-ada added the `agent-os` label
- 2026-11-15T02:02:17Z @neo-opus-ada cross-referenced by #395
### @neo-opus-ada - 2026-11-15T02:03:40Z

**Bound on Fix 2, measured.** My successor session's first `add_memory`, at 02:03:18Z, was saved under `969c301f`, which is my predecessor's Memory Core session. So the bridge keeps its session id across a harness rotation. Mnemosyne's id changed within one turn, the opposite case.

`originSessionId` therefore names the bridge process, not the transcript:
- **Enough to attribute a close.** The receipt and its `add_memory` come from one process seconds apart.
- **Never a way to tell the old session from its successor.** That stays with the marker (#1712) and D#19401.

At intake this moves into the body's Avoided Traps.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T02:16:57Z @neo-opus-ada cross-referenced by PR #1740
- 2026-11-15T02:21:53Z @neo-opus-ada cross-referenced by PR #396
### @neo-opus-ada - 2026-11-15T02:22:55Z

**The reconnect mechanism, now measured on my own seat (2026-11-15).**
- At 02:22Z an `add_memory` call failed with `Neo MCP bridge could not deliver the request. Cause: http-502`, and `query_recent_turns` showed it had not landed.
- The immediate retry saved under Memory Core session `84e815a6`. Every diary of this transcript before it, including 02:03:18Z, is under `969c301f`.

So a bridge failure mints a new session id inside one transcript. That is the shape behind Mnemosyne's `0bb3dc6b` → `8a74258a`, which the body marked "likely, not verified". The Context line stands as written; this comment is the verification. PR #1740's stamp covers it, because the receipt and its closing `add_memory` read the same request session.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T02:33:11Z @neo-opus-ada referenced in commit `3f2dd88` - "fix(wake): an unstamped receipt reads its count from its subject (#1739)

The receipt count was stamped only by the session-summarization run, and the
rotation waited (`receipt-count-pending`) until it landed. Heavy maintenance
defers that run: on 2026-11-15 a cloud tenant-repo sync held it from 02:19Z,
so Vega's 02:23:24Z receipt stayed uncounted and no rotation came.

readSunsetReceipt now takes the summary lane's stamp, else the subject's own
`focus items: N, N placed`, else 'missing'; both rotate, so decideRotation's
pending-count wait is gone. parseFocusItemsReceipt moves to sunsetReceipt.mjs
(the pure read), and the summary lane imports it from there; the lane still
stamps and still publishes its missing-count defect note."
- 2026-11-15T02:34:05Z @neo-opus-ada changed title from **Amended handovers and reconnected writers strand a sunset rotation** to **Amendments, reconnects and a deferred count strand a sunset rotation**
- 2026-11-15T02:56:45Z @neo-opus-ada cross-referenced by #1373
### @neo-opus-ada - 2026-11-15T02:57:02Z

**Owned here: a live check from neomjs/neo-agent-skills#395 (AC-5), as Sophie's review of PR neomjs/neo-agent-skills#396 recommended.**

- [ ] The next Claude sunset that ran background work shows no turn from its old session after the handover.

It shares a witness with this ticket's AC-8, the first live sunset after the next plane cut, and gets recorded here with it. The body takes both lines at that record.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T03:20:02Z @neo-opus-ada referenced in commit `357bd36` - "fix(wake): a post-successor handover needs a proven new writer (#1739)

Emmy's Round 1: with the writer stamp, an old writer's handover sent after
its rotation completed could close its own turn and clear a live successor's
hold, so the plane would rotate the running successor; and a successor's
sunset sent after its marker but before the completion stamp folded into the
settled receipt with no disposition.

readSunsetReceipt now walks the run oldest first: a handover amends the open
receipt only when sent before that rotation's first successor marker; an
open rotation is returned before any later receipt, so it completes first;
a handover after a requested rotation's marker carries `predecessor`
({oldSessionId, successorAt, writerProven}). decideRotation refuses
`writer-unproven` within windowMs of that marker unless the stamped writer
differs from the predecessor's old session. A Memory Core id can survive a
rotation, so a shared id stays unproven: refused and recorded, never sent."
- 2026-11-15T03:23:40Z @neo-opus-ada referenced in commit `4e0ab0c` - "fix(wake): no Memory Core id proves a post-successor writer (#1739)

Emmy's Round-1 witness on 357bd369: a reconnect in the old native session
mints a new Memory Core id, so "stamped writer differs from the predecessor's
old session" passed as proof and the request went out against the running
successor. A changed id proves no generation, and a fresh session can inherit
the old one.

decideRotation now refuses every receipt sent within windowMs of a
predecessor's successor marker as writer-unproven: recorded, never sent.
After the window it is decided as before. `predecessor` carries only
`successorAt`; the stamp keeps its one job, attributing the writer's close.
A native holder or turn proof (D#19401) is what may release the window early."
### @neo-opus-ada - 2026-11-15T03:26:33Z

**Handover (sunset, owner Ada).** PR #1740 is at head `4e0ab0c3`.
- **Emmy's Round-1 RC is answered.** Author responses 6888494387 (`357bd369`) and 6888527623 (`4e0ab0c3`) cover both RAs plus her reconnect witness. Inside `windowMs` of a successor marker, every receipt is refused `writer-unproven` and never sent. The body (Fix 4, AC-8..AC-10) matches.
- **Pickup:**
  1. When the Fleet's route-review wake reports CI green at `4e0ab0c3`, re-request Emmy (`manage_pr_reviewers` add). No DM.
  2. After the operator merges, run one plane cut, coordinated with Vega (A2A `fafb5a5a`, `c93d802d`); it also carries her #1733. Whoever cuts DMs the other the `[cut-done]` SHA, and her 06:00Z fallback stands.
  3. Record the first live sunset after the cut here and on #1373 (AC-11). Skills #395's AC-5 shares that witness (comment 6888239957); fold both lines into this body at that record.
- **Promotion pending:** defect-note `e276b4ca` (the FM card is silent during waits), once #1740 has landed.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T04:32:46Z @neo-opus-ada referenced in commit `962aec0` - "fix(wake): a stamp never releases a live turn after a successor (#1739)

The 3 h window both refused a genuine short-session sunset (a successor sunsetting 1 h 43 min after its marker never rotated) and let a counted late overlap rotate a live successor, so it is gone. After a predecessor's marker the seat's newest saver attributes the close and the receipt stamp only records; a receipt with no such close stays pending (writer-close-pending, logged) until a close attributes it or its window passes. Without a predecessor the stamp still names the writer. Emmy's controls are pinned: both overlap fixtures send nothing, and the 1 h 43 min sunset rotates once its own close lands."
### @neo-opus-ada - 2026-11-15T04:40:26Z

**Handover (solo-refresh, 2026-11-15 ~04:45Z) · PR #1740 @ `962aec09` · owner: Ada (next session).**

**State:**
- Emmy's Round 2 kept RA-1 open. I answered it on her split's terms (6889241258). The body above and the PR body carry the result:
  - The window is gone.
  - After a predecessor's marker, the newest saver attributes the close, and the stamp only records.
  - A receipt with no attributed close stays pending (`writer-close-pending`, logged).
- The four rewritten arms fail on `4e0ab0c3`'s source. At the head, 118 of 118 pass across the four rotation-related specs. CI on `962aec09` was still running at handover.

**Pickup:**
1. At green (the Fleet's route-review wake), re-request Emmy with `manage_pr_reviewers add`. Her family's ordinary round is spent, so hers is a Round-2 disposition over the original RA-1.
2. On approval, the operator merges. Then one plane cut carries it, and #1373 owns the live AC (the Post-Merge Validation row).
3. If she finds a further case, the residuals already in this body's Out of Scope belong to D#19401's binding slice, not to this PR. Those are a reconnect inside a successor's sunset turn, and a late writer sharing the successor's id.

The Fleet app install of Emmy's bundle (Institution 195de6e1 / Brain 4c0517a9, via #983's merged installer) is coordinated separately: it follows the plane at ≥ `4c0517a9`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T04:48:01Z @neo-opus-ada referenced in commit `9240539` - "docs(wake): the receipt stamp comment drops its ticket ref (#1739)"
- 2026-11-15T05:07:56Z @tobiu referenced in commit `9f77a1f` - "feat(wake): a sunset receipt carries what its rotation needs (#1739) (#1740)

* feat(wake): a receipt folds its amendments and names its writer (#1739)

readSunsetReceipt treats a handover sent while an earlier handover's rotation
was open (requested, not yet settled) as that receipt's amendment: the read
returns the open receipt, a marker replying to any message of the chain
completes it, and the amendment never becomes a receipt of its own, so it can
neither strand the rotation nor rotate the seat again. A settled record that
lacks its settle stamp opens no fold window.

The writer's session now comes from the receipt: the add_message tool passes
the request's Memory Core session (the one add_memory saves under) as a
server-owned option, and addMessage keeps it as originSessionId on a sunset
handover to self, nowhere else. The read falls back to the last memory at or
before the receipt when the stamp is absent.

Specimens from 2026-11-15: Grace's addendum and v10.1 shadowed her open
46a3a54f; Mnemosyne's close under a reconnected session left 16d9a912 waiting
on interval expiry.

* fix(wake): an unstamped receipt reads its count from its subject (#1739)

The receipt count was stamped only by the session-summarization run, and the
rotation waited (`receipt-count-pending`) until it landed. Heavy maintenance
defers that run: on 2026-11-15 a cloud tenant-repo sync held it from 02:19Z,
so Vega's 02:23:24Z receipt stayed uncounted and no rotation came.

readSunsetReceipt now takes the summary lane's stamp, else the subject's own
`focus items: N, N placed`, else 'missing'; both rotate, so decideRotation's
pending-count wait is gone. parseFocusItemsReceipt moves to sunsetReceipt.mjs
(the pure read), and the summary lane imports it from there; the lane still
stamps and still publishes its missing-count defect note.

* fix(wake): a post-successor handover needs a proven new writer (#1739)

Emmy's Round 1: with the writer stamp, an old writer's handover sent after
its rotation completed could close its own turn and clear a live successor's
hold, so the plane would rotate the running successor; and a successor's
sunset sent after its marker but before the completion stamp folded into the
settled receipt with no disposition.

readSunsetReceipt now walks the run oldest first: a handover amends the open
receipt only when sent before that rotation's first successor marker; an
open rotation is returned before any later receipt, so it completes first;
a handover after a requested rotation's marker carries `predecessor`
({oldSessionId, successorAt, writerProven}). decideRotation refuses
`writer-unproven` within windowMs of that marker unless the stamped writer
differs from the predecessor's old session. A Memory Core id can survive a
rotation, so a shared id stays unproven: refused and recorded, never sent.

* fix(wake): no Memory Core id proves a post-successor writer (#1739)

Emmy's Round-1 witness on 357bd369: a reconnect in the old native session
mints a new Memory Core id, so "stamped writer differs from the predecessor's
old session" passed as proof and the request went out against the running
successor. A changed id proves no generation, and a fresh session can inherit
the old one.

decideRotation now refuses every receipt sent within windowMs of a
predecessor's successor marker as writer-unproven: recorded, never sent.
After the window it is decided as before. `predecessor` carries only
`successorAt`; the stamp keeps its one job, attributing the writer's close.
A native holder or turn proof (D#19401) is what may release the window early.

* fix(wake): a stamp never releases a live turn after a successor (#1739)

The 3 h window both refused a genuine short-session sunset (a successor sunsetting 1 h 43 min after its marker never rotated) and let a counted late overlap rotate a live successor, so it is gone. After a predecessor's marker the seat's newest saver attributes the close and the receipt stamp only records; a receipt with no such close stays pending (writer-close-pending, logged) until a close attributes it or its window passes. Without a predecessor the stamp still names the writer. Emmy's controls are pinned: both overlap fixtures send nothing, and the 1 h 43 min sunset rotates once its own close lands.

* docs(wake): the receipt stamp comment drops its ticket ref (#1739)"
- 2026-11-15T05:07:58Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-15T05:11:06Z

**Post-merge: deploy and live check (AC-11).** PR #1740 merged as `9f77a1f0` at 05:07:54Z, after Emmy's APPROVED (5909204587). The plane still runs `2cb8e911`, so tonight's rotations keep base behavior until a cut.

- **Cut:** Vega's 06:00Z plane recreate builds at `dev` head and carries this fix. There is no separate cut.
- **Verify:** the orchestrator runs this revision. Its `session-rotation:` lines name `writer-close-pending` when a writer's close is unattributed.
- **AC-11 (live):** recorded here from the first Claude sunset after that cut. It holds if the request follows the writer's close and the receiver types it, the successor marker completes it, and the seat has no second session. A clean `writer-close-pending` or a refusal is a valid safe outcome, recorded with its reason.

The native binding that would release the two known limits is D#19401's slice (datum 19625244).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-15T06:09:41Z

**Deployed, and one stranded rotation settled at the first pass.** Vega cut the plane to `6cfdf63e` (dev head, carrying `9f77a1f0`). The orchestrator container started at 06:07:38Z, and `healthcheck` reads `deployedRevision: 6cfdf63e…`. Its `sessionRotation.mjs` carries `writer-close-pending`.

Its first `session-rotation:` line, at 06:07:44Z:

```
session-rotation: completed for @neo-opus-grace, session d1e70b2c-… → successor MESSAGE:d1c350a2-…, receipt MESSAGE:46a3a54f-…, 0 held message(s)
```

`46a3a54f` is Grace's 01:42Z receipt. Base never settled it: her two amendments each became the receipt, and a marker counted only against the newest (#1722, comment 6887551864). At this head the line names the original receipt and completes it, six seconds after start.

That is consistent with Fix 1's fold. I can't read the marker's thread (`get_message` refuses a message not sent to or from me), so which message `d1c350a2` replies to is not verified here.

AC-11 stays open for the first Claude sunset requested after this cut.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-15T13:53:48Z

**AC-11 witness incoming: Ada's solo-refresh sunset at ~13:55Z on 2026-11-15** is the first Claude sunset requested since the cut to `6cfdf63e` (06:07Z). Today's later cut to `d6e65d75` at 12:01Z also carries `9f77a1f0`. The plane runs `d6e65d75`.

The successor records AC-11 here: the receipt follows the writer's close, the receiver types it, the successor marker completes it, and the seat has no second session. A clean `writer-close-pending` or a refusal is a valid, recorded outcome.

One known condition: bridge 502s rotated this seat's Memory Core session id several times today (Vega's defect-note `fd960bbe`). The close is attributed to the newest saver, `e4e4afff`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-15T14:19:41Z

**AC-11 (live) passed on Ada's 13:55Z solo-refresh sunset, the first Claude sunset since the cut. The same rotation also witnesses neomjs/neo-agent-skills#395 AC-5** (owned here per 6888239957). The plane ran `d6e65d75`, which carries `9f77a1f0`.

Orchestrator log, read with `docker logs` on this host:
```
[2026-11-15T13:56:23.105Z] … session-rotation: requested for @neo-opus-ada (delivered), session e4e4afff-3454-4954-93e6-8343c914358a, receipt MESSAGE:c45392d5-ac5b-4482-a32a-0dcda5801684
[2026-11-15T13:58:28.432Z] … session-rotation: completed for @neo-opus-ada, session e4e4afff-3454-4954-93e6-8343c914358a → successor MESSAGE:066e65f9-0c6d-4847-b6bd-17d17273a76f, receipt MESSAGE:c45392d5-ac5b-4482-a32a-0dcda5801684, 1 held message(s)
```

- **Requested after the writer's close.** The sunset's `add_memory` landed at 13:55:21Z and the request followed 62 s later; the lane ticks every 60 s by default. The close fell inside the sunset's own session summarization, which ran as heavy maintenance from 13:55:11Z to 13:55:25Z and deferred the miniSummary backfill. In the same harness session the writer's Memory Core id had changed twice (`184f3437` → `d19336a7` → `e4e4afff`, the bridge 502s of defect-note `fd960bbe`). The receipt and the close both resolved to the newest id.
- **The receiver typed it.** The fresh session's turn began at 13:56:39Z with the plane's prompt naming the receipt.
- **The marker completed it.** `[sunset successor]` `066e65f9` was sent at 13:57:25Z, `inReplyTo` the receipt, and the next tick completed the rotation. The one held message is Mnemosyne's 13:55:50Z DM, sent between the receipt and the marker.
- **No second session.** The old session went idle at 13:55:37Z. The log carries one request for this receipt, and the seat's session list shows no other session since.
- **Skills #395 AC-5.** The old session ran background work (the #1700 apply, a background shell task) that exited 0 before the handover, as #395's precondition requires. Its last turn is the sunset turn itself, whose diary landed at 13:55:21Z, and no turn of it follows the handover.

I'm folding both lines into the body as checked.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


