---
id: 1722
title: A sunset rotates late or never; a late rotation opens a second session
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T18:56:37Z'
updatedAt: '2026-11-15T01:51:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1722'
author: neo-opus-ada
commentsCount: 4
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
closedAt: '2026-11-15T00:09:58Z'
---
# A sunset rotates late or never; a late rotation opens a second session

## Context

Design authority: the operator's ruling, 2026-11-14 ~22:1xZ: *"a sunset should be terminal. session close and start a fresh session instantly. this 'hey let us wait another 2 minutes' is not a benefit, but a failure. this just increases the chance to receive MORE a2a messages."* A rotation that opens a second live session for one seat *"violates 'never a copy'"*. #1383's guards say the same at request time: a session the operator drives is never rotated, and a seat already restarted is not rotated again.

Four Claude sunsets on 2026-11-14 produced no rotation:

| receipt | sunset | what happened |
|---|---|---|
| Ada `798b55dc` | 18:49:04Z | never requested: two older intervals that never closed held `live-turn` until 19:45Z (this ticket's first finding) |
| Grace | 21:04:02Z | requested, then the receiver failed in 10 s: `Target app lost frontmost status after activation (-2700)`; terminal, no retry |
| Ada `13868975` | 21:54:16Z | requested 21:54:52Z, accepted by the receiver, then deferred 9× for `human-input-active` and typed at 22:03:55Z. The operator had started a fresh session by hand at ~22:00Z, so this opened **a second live session** of the seat. The operator stopped it at 22:04:23Z; it had run reads only. |
| Mnemosyne `03c5b7f5` | 22:04:52Z | refused at 22:05:16Z, `prompted-after-receipt`. Nobody prompted her session. |

Instruments: read-only `better-sqlite3` reads of `AGENT_TURN_PRESENCE:@<seat>:*` and the receipt nodes on the plane's graph store, ~22:10Z. The orchestrator log `session-rotation:` lines. The receiver's record `session-rotate:MESSAGE:13868975…` (`deferCount: 9`, `state: delivered`, `dispatchFinishedAt: 22:04:02.934Z`) and its `launchd.err.log`.

## The Problem

**1. The plane reads the receipt turn's own presence as a live turn or a prompt.** One Claude turn leaves several intervals with unrelated `turnId`s, and `add_memory` closes only the one it closes. Mnemosyne's sunset turn:

| interval | source | started | last progress | status |
|---|---|---|---|---|
| `…f987b1212c5c` | `claude-user-prompt-submit` | 21:59:33.880Z | 21:59:33.880Z | `active` (never closed) |
| `…de7d0ce6d745` | `claude-post-tool-use` | 22:03:10.879Z | 22:04:52.763Z | `active` (never closed) |
| receipt sent | | 22:04:52.534Z | | |
| `…36c1796f327c` | closed by `add_memory` | **22:04:52.814Z** | 22:06:14.987Z | `terminal` |

- `promptedTurnAt` is the earliest interval started after the receipt without a wake nonce. The third interval opened 280 ms after the receipt, inside the same turn, so the turn refused its own rotation.
- My 21:54Z turn passed only because its closing interval had opened at 21:52:17Z, before its receipt. Whether a sunset rotates depends on call order.
- `liveTurnUntil` counts unclosed intervals started by the receipt's time. The never-closed hook intervals hold it until they expire (an hour). The progress rule in PR #1726 would still hold Mnemosyne's receipt to 23:04:52Z through `…de7d0ce6d745`. So #1726 is withdrawn.

**2. The receiver types a rotation long after the guards were read.** The plane checks the guards once, before it sends. The receiver then waits for 60 s of keyboard and pointer quiet (`HUMAN_INPUT_QUIET_MS`), re-asked once per 60 s re-drain, for up to 120 deferrals (`HUMAN_INPUT_DEFER_BOUND`, about 2 h). It types without re-reading anything. #1383 assumed *"a late session is not a lost one"*; the 21:54Z receipt shows a late session is a copy.

**3. An activation failure before any keystroke ends the rotation.** Grace's script failed in its activation phase, before any keystroke. That is safe to retry, but it is terminal today.

## The Architectural Reality

- `ai/daemons/wake/sunsetReceipt.mjs` `readSunsetReceipt`: the `promptedTurn` and `liveTurn` queries. `decideRotation` and the wake hold (`holdsWakes` → `isReceiptOpen`) consume both. A false `promptedTurnAt` also releases the wake hold early.
- `ai/daemons/wake/sessionRotation.mjs`: `HUMAN_INPUT_QUIET_MS`, `HUMAN_INPUT_DEFER_BOUND`, `evaluateHumanInput`.
- `ai/daemons/wake/receiver.mjs`: the drain's `deferred` branch, the `drainIntervalMs` timer, and `STEP_BOUNDS_MS.dispatch` (30 s over the route's 10 s `attemptTimeoutMs`).
- `ai/daemons/wake/localWakeAdapters.mjs`: the rotation's human-input read, then `buildOsascriptArgs`. Its activation loop asserts `"after activation"` before the first keystroke.
- The presence writers (`turnPresenceHook`, `TurnPresenceService`) stay as they are. The read has to be right about the intervals they already write.

## The Fix

1. **The receipt turn ends at its writer's close.** `readSunsetReceipt` reads the close as the earliest `terminal` interval at or after the receipt that the writer's own `add_memory` lands with: a memory of the writer's own Memory Core session within `WRITER_CLOSE_SKEW_MS`. Another close authorizes nothing, and so does one saved under a session first seen after the receipt: that may be a reconnect or a new session, so the turn waits on presence alone. *(Amended after Emmy's Round 1: a seat-wide terminal timestamp cannot tell the writer's close from another session's.)*
   - Until the close exists, the turn is live, bounded by its intervals' expiry as today.
   - Once it exists, nothing earlier holds the request.
   - `promptedTurnAt` is the earliest of three things: a wake-nonce-free interval started after the receipt that the writer's close did not close; an interval that progressed after the close; or a memory after the receipt from a session already writing before it. Each is a person or another session. Intervals the writer's close closed are its own tail.
2. **A rotation types at once or never.**
   - The receiver fails a rotation once `ROTATION_MAX_AGE_MS` (60 s) has passed since it accepted it: `session-rotate-expired`, and it never types.
   - Inside that age it types at the first `HUMAN_INPUT_QUIET_MS` = 3 s pause. A deferred rotation is re-asked about once a second, not once per 60 s re-drain.
   - The age bound replaces `HUMAN_INPUT_DEFER_BOUND`.
3. **An activation failure before the first keystroke defers** within the same age. A failure in any later phase stays terminal, because Cmd+N may already have opened a session.
4. **The fresh-session keystroke is fenced where it happens** *(Emmy's Round 1)*. None of these opens a session:
   - an attempt the receiver stopped waiting for (its abort signal);
   - a deadline crossed during the pid, dialog or idle reads;
   - a seat whose holder in the Claude wake listener's record changed since the rotation's first attempt.

   The script re-checks the deadline right before Cmd+N.

## Acceptance Criteria

- [ ] Unit (orchestrator spec), red on `dev` first: Mnemosyne's three-interval shape is requested at the head, where `dev` refuses it `prompted-after-receipt`.
- [ ] Unit: an interval started after the close refuses the request (`prompted-after-receipt`), and so does one between the receipt and the close that the writer's close did not close. One it did close does not refuse. Another session's close, while the writer is active, requests nothing.
- [ ] Unit: older intervals that never closed, plus a closed receipt turn, let the request go (this ticket's first finding). A receipt turn with no close yet waits.
- [ ] Unit (receiver spec): a rotation deferred for human input is re-asked within about a second. A rotation past `ROTATION_MAX_AGE_MS` fails `session-rotate-expired` without typing. A quiet reading of 3 s types. None of these types the fresh-session keystroke: a deadline crossed mid-attempt, an attempt that outlives its adapter timeout, or a session opened by hand during a defer.
- [ ] Unit (adapter spec): an `"after activation"` failure on a rotation returns `deferred`; a later-phase failure returns `failed`.
- [ ] Post-merge (live): the next Claude sunset is requested within one lane tick of its `add_memory`, typed within 60 s, and completed by its successor marker. No second session of that seat exists.

## Out of Scope

- One `turnId` per turn across the hook and `add_memory` writers (the writer-side fix). The read must be right about the intervals that exist.
- The plane learning the receiver's terminal outcome. The window still turns an undelivered rotation `incomplete`.
- Non-Claude seats: only Claude Desktop seats get a `SESSION_ROTATE` route.

## Avoided Traps

- **A self-clear (`clear_session("self")`).** The operator ruled it out on 2026-11-05 (neomjs/neo-agent-skills#274). #1383's 11-02 data shows a cleared session goes dark: no `Stop` hook, no listener, and a `ScheduleWakeup` does not survive the clear.
- **The `claude://code/new?q=` deep link.** It only pre-fills the prompt (#1682).
- **Typing with no quiet check at all.** The activation would take focus from a person mid-keystroke and route their text into the seat's new composer. A 3 s pause keeps "instant" without that.
- **Longer windows or more retries.** Every wait is time for more wakes to reach a session that has ended.

## Related

PR #1726 (withdrawn: its progress rule holds hook-written receipts for an hour) · #1383 · #1682 · #1712 / PR #1720 · #1713 / #1714 · #1373 · neomjs/neo-agent-skills#274

Live latest-open + closed sweep: latest 20 open and a closed search (rotation sunset fresh session; Out of Scope / Avoided Traps read on #1383 and #1682) at 2026-11-14T22:2xZ. No equivalent. Backlog 157 open (red), so this folds into the owning ticket rather than filing a new one. A2A: the latest 30 messages, no claim on rotation. MC: one problem-noun query, my own lane only. Own assignments: this ticket is the owner.
Labels: not `good first issue` / `help wanted`: it blocks every Claude night rotation, and I own it.

Retrieval Hint: "sunset rotation prompted-after-receipt receipt turn add_memory close turnClosedAt human-input quiet age bound second session copy"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-14T18:56:37Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T18:56:39Z @neo-opus-ada added the `bug` label
- 2026-11-14T18:56:39Z @neo-opus-ada added the `ai` label
- 2026-11-14T18:56:39Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T19:33:39Z @neo-opus-ada cross-referenced by PR #1726
- 2026-11-14T21:37:58Z @neo-opus-ada referenced in commit `b27907f` - "fix(wake): progress, not start order, names the receipt's turn (#1722)

The newest interval started by the receipt's time is not proof of the turn
that wrote it: two sessions of one seat can overlap, so a later turn of the
other session that closed before the receipt read as the receipt's turn, and
the rotation was requested while the writer was still active.

The receipt's turns are now the intervals started by its time. Those that
progressed at or after the receipt were alive when it was written, and when any
did, only they count, so an older interval that went quiet before the receipt
holds nothing. Without such progress, every interval started by then counts, as
before."
### @neo-opus-ada - 2026-11-14T21:53:45Z

**Handover (sunset, owner Ada).**

- **PR #1726:** Emmy's Round-1 action addressed at `b27907fe`, required CI green, Emmy re-requested at 21:48Z.
  - Her case: two sessions of one seat overlapping, with a newer turn of the other session closed before the receipt.
  - The fix: intervals that progressed at or after the receipt are the receipt's turns; without such progress, every interval begun by then counts.
  - The same specs fail at `c93e8a0e` (overlap) and at `e4bc5491` (serial), and pass at the head. Response: [6884608962](https://github.com/neomjs/neo-agent-brain/pull/1726#issuecomment-6884608962).
- **After the merge:** one plane cut to the merge SHA, run by me. `dev` also carries #1724 and #1727 above the live `e4bc5491` (#1721 is an npx package), so the cut covers them. I coordinate with Emmy and Vega so exactly one cut runs.
- **AC-3:** record here the first Claude sunset after that cut which follows a diary-less turn and still requests within one lane tick (Residual-Owner #1373).
- **The sunset writing this comment is a live test of the plane as deployed (`e4bc5491`).** That plane reads the turn-presence intervals the old way, but none can hold the rotation now: the two orphans from the 18:5xZ session expired at 19:42Z and 19:45Z. The request should go at the first lane tick after this turn's `add_memory` closes its interval. Whatever happens (request, delivery, fresh session, successor marker, completion) gets recorded here and on #1373.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-14T22:22:19Z @neo-opus-ada changed title from **An older turn that never closed holds a sunset's rotation for an hour** to **A sunset rotates late or never; a late rotation opens a second session**
- 2026-11-14T22:33:39Z @neo-opus-ada cross-referenced by PR #1730
- 2026-11-14T23:22:52Z @neo-opus-ada referenced in commit `3fb98c1` - "fix(wake): a rotation's close and keystroke answer to its writer (#1722)

A terminal interval authorizes a rotation only when the writer's own
add_memory lands with it, under the writer's Memory Core session or one
first seen after the receipt. Another close authorizes nothing. After
the writer's close, an interval opened after the receipt that the close
did not close, activity after the close, or a memory from a session
already writing before the receipt refuses the request.

The fresh-session keystroke is fenced where it happens. An attempt the
receiver stopped waiting for, a deadline crossed during the pid, dialog
or idle reads, or a seat another session claimed since the first
attempt opens no session. The script re-checks the deadline right
before Cmd+N."
- 2026-11-14T23:43:27Z @neo-opus-ada referenced in commit `e3981cf` - "fix(wake): only the writer's session closes the receipt turn (#1722)

A session first seen after the receipt no longer attributes a close:
it may be the writer's reconnect or a new session, so the turn waits on
presence alone. A seat holder that appears after an unreadable first
read supersedes the rotation, and the script re-reads the holder right
before Cmd+N, refusing the keystroke when the record names another
session or cannot be read."
- 2026-11-14T23:54:39Z @neo-opus-ada referenced in commit `33ef3ec` - "fix(wake): an absent seat holder stays expected at the shortcut (#1722)

With no holder record at a rotation's first attempt, the receiver hands the adapter the expected absence, and the script requires the record still absent right before Cmd+N, so a session opened during activation is seen."
- 2026-11-14T23:59:21Z @neo-opus-ada referenced in commit `0e8a8d1` - "fix(wake): the shortcut fence compares the seat's owner, not any session it names (#1722)

The listener record also names the session a wake was delivered to, so a contains-check passed when a new owner held the seat. The script reads owner.sessionId with plutil and requires it to equal the first attempt's holder; absence is checked with test -e."
- 2026-11-15T00:09:58Z @tobiu referenced in commit `2cb8e91` - "fix(wake): a sunset rotates at once, never late (#1722) (#1730)

* fix(wake): a sunset rotates at once, never late (#1722)

The plane reads the receipt's turn as ending at its close, the first
terminal interval after the receipt. Before it, the turn is live and an
interval it opened after the receipt is its own tail, not a prompt.
After it, older intervals that never closed hold nothing, and only
activity started after the close refuses the request.

The receiver types a rotation at a 3 s pause, re-asked about once a
second, or fails it once 60 s have passed since its accept. A late
session is a second live session of a seat restarted by hand. An
activation that loses the front before the first keystroke typed
nothing and is asked again inside that age.

* fix(wake): a rotation's close and keystroke answer to its writer (#1722)

A terminal interval authorizes a rotation only when the writer's own
add_memory lands with it, under the writer's Memory Core session or one
first seen after the receipt. Another close authorizes nothing. After
the writer's close, an interval opened after the receipt that the close
did not close, activity after the close, or a memory from a session
already writing before the receipt refuses the request.

The fresh-session keystroke is fenced where it happens. An attempt the
receiver stopped waiting for, a deadline crossed during the pid, dialog
or idle reads, or a seat another session claimed since the first
attempt opens no session. The script re-checks the deadline right
before Cmd+N.

* fix(wake): only the writer's session closes the receipt turn (#1722)

A session first seen after the receipt no longer attributes a close:
it may be the writer's reconnect or a new session, so the turn waits on
presence alone. A seat holder that appears after an unreadable first
read supersedes the rotation, and the script re-reads the holder right
before Cmd+N, refusing the keystroke when the record names another
session or cannot be read.

* fix(wake): an absent seat holder stays expected at the shortcut (#1722)

With no holder record at a rotation's first attempt, the receiver hands the adapter the expected absence, and the script requires the record still absent right before Cmd+N, so a session opened during activation is seen.

* fix(wake): the shortcut fence compares the seat's owner, not any session it names (#1722)

The listener record also names the session a wake was delivered to, so a contains-check passed when a new owner held the seat. The script reads owner.sessionId with plutil and requires it to equal the first attempt's holder; absence is checked with test -e."
- 2026-11-15T00:09:59Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-15T00:12:04Z

**Handover: deploy and live check (post-merge, AC-6).** PR #1730 merged as `2cb8e911`. The plane and the host receiver still run `e4bc5491`, so tonight's sunsets keep the old behavior until a cut.

Pickup, owner: Ada's next session, 2026-11-15:
1. **Cut.** One plane cut at the merge SHA: `build.sh`, then `cut.sh`, which needs the operator's one click. `dev` above `e4bc5491` also carries #1724, #1727 and #1728, so announce `[cutting]` and run exactly one cut. The cut restarts the receiver, the same way its start time matched the 19:11Z cut.
2. **Verify** that the orchestrator runs the new `readSunsetReceipt` and the receiver loaded the fence: its log shows `rotation DEFERRED … asked again in 1000 ms` on a deferral.
3. **Live AC-6:** the next Claude sunset. Record request → typed within 60 s → successor marker → `completed`, and no second session of the seat. A refusal or expiry is a valid safe outcome; record its reason.

The residual is in defect-note `e3ad54e4`: the receipt names no sender session.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-15T01:40:58Z

**Live AC in flight (2026-11-15 ~01:45Z).** #1730 has been deployed since the plane cut to `2cb8e911` at 00:28:40Z. Ada's own sunset now, at the operator's request, is the first Claude sunset since then, and it is the live witness.

The successor session records the outcome here as its first act, read from three sources:
- the orchestrator's `session-rotation:` lines: requested, then completed or refused;
- the receiver's `launchd.err.log`: DEFERRED, EXPIRED or SUPERSEDED lines, or a typed rotation;
- the `sunset-successor` marker.

The AC holds if the sunset is typed within 60 s of the request and completed by its successor, with no second session. A clean refusal or an expiry is recorded with its reason.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-15T01:51:10Z

**Live AC passed on Ada's seat (2026-11-15, plane `2cb8e911`).** Grace's seat rotated in the same tick and surfaced one residual.

| | Ada | Grace |
|---|---|---|
| receipt | `34df17be`, 01:42:02Z | `46a3a54f` |
| final `add_memory` → request | 01:42:14.6Z → 01:42:44.9Z (+30 s, one tick) | → 01:42:44.95Z |
| typed (receiver record `delivered`) | 01:42:55.3Z (+10.4 s) | 01:43:03.2Z (+18.2 s) |
| successor marker → orchestrator | `86716014` 01:43:44Z → `completed` 01:43:47Z | none matched (below) |
| second session | none: the old session has been idle since 01:42:32Z | fresh session confirmed by the operator |

For either rotation, the receiver logged no DEFERRED, EXPIRED or SUPERSEDED line. Its only line was `dialog gate could not probe … delivering fail-open`.

**Residual (Grace's seat).** After her rotation was typed, the seat sent two more `sunset-protocol-handover` self-DMs: an addendum without focus items (`ac5bc324`), then a corrected v10.1 (`e2a6f73c`). Her defect-note `69793cf1` names that cause. `readSunsetReceipt` reads only the newest non-marker self-DM, so:
- each amendment became the seat's receipt, and each was refused `minimum-interval` (01:45:03Z, 01:47:07Z);
- the in-flight `46a3a54f` rotation can no longer complete, because a marker counts only when it replies to the newest receipt. The plane never settles it;
- only the 30-minute `minIntervalMs` stopped a second rotation. Sent after that interval, the same amendment is requested as a new rotation, against whichever session then holds the seat.

Still open: which session sent the amendments. If it was the old one, something woke it after its sunset. The operator notes it had run several subagents, and a background subagent's completion re-invokes its parent session. Once the receiver has the request, `decideRotation` only waits for a marker and no longer reads the old session's turns.

#1373's AC-6 is a different contract (`dispatch-restored` vs `agent-resumed`), so nothing is claimed there.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


