---
id: 1770
title: 'A sunset rotation expires while the host is busy, and nobody is told'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-15T20:54:48Z'
updatedAt: '2026-11-15T23:57:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1770'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-11-15T23:34:39Z'
---
# A sunset rotation expires while the host is busy, and nobody is told

## Context

The operator asked why `@neo-fable-mnemosyne`'s sunset at 19:43Z on 2026-11-15 opened no fresh session, and later why `@neo-opus-ada`'s 22:28Z sunset opened none. Five sunsets ran that day:

| Receipt | Seat | Plane (`session-rotation`) | Host receiver record | Result |
|---|---|---|---|---|
| 17:35Z | `@neo-opus-ada` | requested (delivered) 17:35:34Z | typed | `completed` 17:37:39Z |
| 19:27Z | `@neo-opus-ada` | requested (delivered) 19:27:50Z | accepted 19:27:50.125Z · `deferCount` 5, `activation-lost` · `failed` `session-rotate-expired` 19:29:00.314Z | no session; the old one was resumed by hand at 20:07Z |
| 19:43Z | `@neo-fable-mnemosyne` | requested (delivered) 19:43:21Z | accepted 19:43:21.836Z · `deferCount` 5, `activation-lost` · `failed` `session-rotate-expired` 19:44:26.538Z | seat dark; last turn 19:42:22Z |
| 20:35Z | `@neo-opus-ada` | requested (delivered) 20:35:54Z | typed | `completed` 20:37:58Z |
| 22:28Z | `@neo-opus-ada` | requested (delivered) 22:28:59Z | accepted 22:28:59.604Z · `deferCount` 10, `human-input-active` · `failed` `session-rotate-expired` 22:30:00.065Z | no session; the operator started one by hand at ~22:46Z |

**Observed (locked screen, 19:27Z and 19:43Z).** Each of the ten failed attempts logged `Target app lost frontmost status after activation (-2700)`, then `rotation DEFERRED … (activation-lost)`. Each rotation ended with `rotation EXPIRED …: not typed within 60000 ms of its accept; failed without a keystroke`. The host's `loginwindow` log shows the screensaver starting at 18:32:26Z and locking the screen in the same millisecond (`startScreenLock: kLWLockFromScreenSaverIdleLaunch`; the 300 s it then reads is the keybag's grace period). There is no stop until 19:52:57Z. At that point it logged `saver is running, start the unlock`, `startUnlock: kLWLockFromScreenSaverIdleLaunch`, and `Keybag is locked, setting promptForPassword = YES`. So the screen was locked from 18:32:26Z to 19:52:57Z, and both lost rotations fell inside that span.

**Observed (a person at the host, 22:28Z).** The screen was unlocked. All ten attempts read `HIDIdleTime` under 3 s and logged `rotation DEFERRED … (human-input-active)` (defer #1 to #10), then `rotation EXPIRED …: not typed within 60000 ms of its accept`. A 20 s sample of the idle clock at 22:54Z climbed from 39 s to 59 s untouched, so nothing synthetic resets it: the input was a person's. A pass took ~5.6 s rather than the 1 s retry, because each drain re-probes the receiver's other pending records first (Out of Scope), so the minute held ten chances to catch a 3 s pause.

**Inferred.** The idle clock (`HIDIdleTime`) reads a locked screen as quiet, so each attempt activates the seat's window. The lock screen keeps the front, and the rotation spends its 60 s on attempts that cannot succeed.

## The Problem

A sunset is lost whenever the host stays busy through the minute after it, and the loss is silent.

1. The receiver gives a rotation 60 s from its accept. A lock lasts longer than that, and so can a person working.
2. The expiry stays on the host. The plane cannot read the receiver's records: its container declares no `fleet.wakeReceiverRecordsDir`, and #1722 left that back-channel out of scope. So the receipt stays `requested` and the plane holds the seat's wakes for `sessionRotation.windowMs` (3 h). The plane then records `incomplete` as one ERROR log line. Until someone notices, the seat is dark and its wakes are held. At 20:5xZ Mnemosyne's receipt was still open.

## The Architectural Reality

- `ai/daemons/wake/receiver.mjs:471` fails a rotation when `isRotationExpired(acceptedAt, now)` (`sessionRotation.mjs:200`) holds, against `acceptedAt + ROTATION_MAX_AGE_MS` (60 s, `sessionRotation.mjs:61`). The holder fence (`receiver.mjs:483` on, re-read by the script right before Cmd+N) fails a rotation once another session claims the seat.
- `localWakeAdapters.mjs` defers `activation-lost` at `:783` and gates typing on `readHumanIdleMs` at `:1265` (`ioreg`, `HIDIdleTime`). Nothing reads the lock state.
- `decideRotation` (`sessionRotation.mjs:354`) answers `wait (receiver-has-request)` for a delivered rotation until `windowMs`, then `incomplete`. `runSessionRotation` (`orchestrator/scheduling/sessionRotation.mjs:87`) only logs it. The plane already publishes defect notes through `Orchestrator#publishDefectNote` (`Orchestrator.mjs:399`), and `DiskPressureDefectService` uses it.
- The console session's lock flag is readable without a privacy grant (`CGSessionCopyCurrentDictionary`, also `ioreg -n Root -d1` → `IOConsoleUsers`). `CGSSessionScreenIsLocked` is absent while unlocked (measured 20:4xZ) and present while locked (AC-1).

(Line numbers are those of `dev` at filing.)

Design authority: `ai/daemons/wake/sessionRotation.mjs:12-18`, "It waits for that pause only {@link ROTATION_MAX_AGE_MS}: a sunset is terminal and its fresh session starts at once, while every minute of waiting lets wakes reach a session that has ended and lets a person restart the seat by hand, after which a late rotation is a second live session of the seat." Neither reason holds under the plane's hold. The plane holds the seat's wakes, so none reach the ended session. A session started by hand, at the host or elsewhere, claims the seat on its first prompt, and the holder fence reads that right before Cmd+N. Measured at 22:46Z: the hand-started successor's first prompt replaced the baseline holder `9e7731f2…#90075` with `702d94b1…#16014`, so a rotation still waiting then would have failed `session-rotate-superseded`.

**Amended after 22:28Z.** This ticket first kept the minute for a person at the host ("the bound measures time spent competing with a person for the front"). The 22:28Z loss falsified that choice. What a person needs protecting from is focus theft, and only an activation takes the front. A deferral on a lock or on input touches nothing, so it has no reason to end the wait.

## The Fix (one PR)

1. **Receiver: under the plane's hold, the hold is the deadline.** Add `readScreenLocked()` beside `readHumanIdleMs` in `localWakeAdapters.mjs`. A locked screen defers `screen-locked` with no activation and no keystroke. A person at the keyboard defers `human-input-active`, as today. Neither ends the wait: `rotationDeadline` is the envelope's `holdUntil`. Lost activations, the one defer that takes the front, are capped at `ROTATION_MAX_ACTIVATION_LOSSES` (5), after which the rotation fails `session-rotate-front-lost`. The 3 s quiet window and the holder fence are unchanged. An unreadable lock state keeps today's path, because the frontmost assertion still guards every keystroke. On this host the screensaver locks the screen as it starts, so the lock flag covers the screensaver too.
2. **Envelope: the receiver never types after the plane has given up.** `buildRotationEnvelope` carries `payload.holdUntil`, the receipt's `sentAt` plus `windowMs`. A wait still open at `holdUntil` fails `session-rotate-expired`. An envelope without the field keeps the 60 s bound from the accept, which covers an older plane.
3. **Plane: `incomplete` is never only a log line.** `runSessionRotation` publishes one `defect-note:` per receipt through `publishDefectNote` when it records `incomplete`, naming the seat, the receipt and the sunset time.
4. **Contract docs.** The `sessionRotation.mjs` header and the `windowMs` JSDoc in `ai/configBase.mjs` (text only; ADR-0019 read before the edit) say the receiver types at the host's first opening, before the hold ends.

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `session/rotate` `payload.holdUntil` | `buildRotationEnvelope` | ISO time the plane's hold on the receipt ends: the rotation's deadline | absent → 60 s from accept | module header | unit |
| defer reasons `screen-locked`, `human-input-active` | `sessionRotation.mjs` constants | the rotation waits for its opening, nothing activated or typed, never past `holdUntil` | lock unreadable → today's path | constant JSDoc | unit + one live lock |
| failure `session-rotate-front-lost` | `ROTATION_MAX_ACTIVATION_LOSSES` | the fifth lost activation ends the rotation | — | constant JSDoc | unit |
| `incomplete` defect note | `runSessionRotation` | one `defect-note:` per receipt | publisher throws → logged, record still written | lane JSDoc | unit |

Decision Record impact: none. No ADR governs rotation. This amends the receiver contract #1722 set in the `sessionRotation.mjs` header.

## Acceptance Criteria

- [x] AC-1: On the operator host, `readScreenLocked()` returns `true` during a real lock and `false` unlocked, each measured once by a sampler that only reads. Receipts go in the PR. (#1771's AC-1 row: the sampler read `true` 3.8 s after loginwindow's `startScreenLock` at 21:46:33Z, and `false` 4.1 s after its `startUnlock` at 21:51:24Z.)
- [ ] AC-2 (unit): a rotation accepted while locked defers `screen-locked` past 60 s with no activation and no keystroke. After the unlock it types at the first 3 s pause.
- [ ] AC-2b (unit): under the hold, a rotation deferred `human-input-active` past 60 s stays pending and types at the first 3 s pause. This is the 22:28Z case, and it must fail on #1771's earlier head `ff723a27`.
- [ ] AC-2c (unit): the fifth lost activation fails the rotation `session-rotate-front-lost` without opening a session, and `human-input-active` defers do not count toward it.
- [ ] AC-3 (unit): a seat that another session claims during the lock fails the rotation `session-rotate-superseded` after the unlock, without a keystroke.
- [ ] AC-4 (unit): a wait still open at `holdUntil` fails `session-rotate-expired` without a keystroke. Without `holdUntil`, the bound stays 60 s from `acceptedAt`.
- [ ] AC-5 (unit): `runSessionRotation` publishes exactly one `defect-note:` when it records `incomplete`. It publishes none on later passes and none for `completed` or `refused`.
- [ ] AC-6 (post-merge, live, after the plane cut and the FM runtime update): the next Claude-seat sunset taken while the host is busy (locked, or in use) opens its fresh session at the host's first opening, and the plane records `completed`. A comment cites the receipt id, the receiver record's defers and the timings.

## Out of Scope

- A receiver → plane back-channel for other failures. The plane still learns of those only at `windowMs`, now with a note.
- The value of `windowMs`. `ROTATION_MAX_AGE_MS` keeps its value, now only for envelopes without a hold.
- The host's lock and screensaver settings.
- Wake delivery during a lock.
- **Observed, not fixed here:** every receiver drain re-probes about 120 pending native (`primary-session`) wake records before a rotation's turn. At 23:1xZ, 100 were `@neo-gemini-julia`'s (oldest accepted 2026-11-14T19:16Z: one `native-recipient:awaiting-readiness`, the rest `earlier native wake outstanding`) and 18 were `@neo-gpt-sophie`'s (oldest 2026-11-15T17:48Z). That is why a rotation pass took ~5.6 s. Reported separately.

## Avoided Traps

- **#1722's "longer windows or more retries".** That trap's reason is wakes reaching an ended session. The plane holds them for the whole wait, and `holdUntil` ends the wait when the hold ends.
- **Pausing the age per busy reason.** The first version counted 60 s of unlocked time, with paused-interval bookkeeping (`lockedMs`, `lockedSince`). Extending it to input would keep the bookkeeping and still expire a rotation whose host stays busy. The hold already bounds the wait, so the bookkeeping goes.
- **Typing through the lock screen or waking the display.** Keystrokes could land in the password field.
- **A blind re-request from the plane.** The plane cannot see the lock, so a timer would race a person's restart.
- **A macOS notification as the alert.** It is a visible surface nobody asked for. The defect note reaches peers through the existing digest.

## Related

#1722 / PR #1730 (the 60 s receiver contract) · #1739 / PR #1740 (rotation residuals) · #1735 (keeps the signed rotation route) · #1373 (sustained wake-delivery repair) · D#19401 (native session binding)

Live latest-open + closed sweep: latest 20 open, plus open and closed search (`rotation`, `screen locked`, `activation-lost`, `session-rotate-expired`, `fresh session sunset`) at 2026-11-15T20:54Z. Out of Scope and Avoided Traps read on #1722 and #1739. #1735 and #1373 are adjacent, not equivalent. Backlog band: 160 open (red); no open owner or epic for rotation.
A2A in-flight sweep: last 30 messages, no rotation or lock claim.
MC sweep: "sunset did not rotate, no fresh session opened, seat stayed dark, screen locked, rotation expired", 6 results, only this investigation and #1722's diaries; no prior decision on a locked screen.
Own-assignment sweep: 4 open, none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): `ai/daemons/wake` and `ai/daemons/orchestrator/scheduling` own the change; no new file.

Labels: not `good first issue` or `help wanted`. It needs the operator host's lock state for AC-1 and AC-6, it amends a contract the operator ruled on, and I own it.

Origin Session ID: fddf8f59-a987-40da-a768-747abdf77ab8

Retrieval Hint: "rotation expired activation-lost locked screen screensaver" · "session-rotate-expired human-input-active no fresh session after sunset"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-15T20:54:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-15T20:54:50Z @neo-opus-ada added the `bug` label
- 2026-11-15T20:54:50Z @neo-opus-ada added the `ai` label
- 2026-11-15T20:54:51Z @neo-opus-ada added the `agent-os` label
- 2026-11-15T21:21:00Z @neo-opus-ada cross-referenced by PR #1771
- 2026-11-15T21:34:35Z @neo-opus-ada referenced in commit `44d3380` - "feat(wake): the incomplete note's hand-start prompt says so (#1770)

The note handed out the plane's own first prompt, which says the plane opened the session. A person who starts the seat from the note would type a false provenance, so the note now offers a prompt that says the session was started by hand after the rotation opened none."
- 2026-11-15T22:14:34Z @neo-opus-ada referenced in commit `dcb0d47` - "fix(wake): lock pause needs the hold; cutoff is absolute (#1770)

Round 1 of the review. Without a valid signed hold, a lock no longer extends the wall-clock minute, so an older plane keeps the 60 s it was promised. The fresh-session script compares an absolute cutoff, read after the holder check and right before Cmd+N, so a child that starts late opens nothing late. The adapter re-reads the lock at its last boundary and after a failed activation, and the receiver opens the paused interval for it. The incomplete note reports only the missing successor marker and asks for the seat's window to be checked before a hand start."
- 2026-11-15T22:23:37Z @neo-opus-ada referenced in commit `ff723a2` - "fix(wake): the front is asserted again right before Cmd+N (#1770)

Round 1, RA-3's second half. A lock that engages after a successful activation took the front before Cmd+N with nothing to catch it. The script now asserts the front right before the cutoff check, and a loss there is retry-safe (isPreSessionFailure): only the tab shortcut, which opens no session, precedes it. The holder and cutoff refusals stay final."
### @neo-opus-ada - 2026-11-15T22:27:14Z

**Handover: sunset, solo-refresh, 2026-11-15**

- **State:** #1771 is at `ff723a27` and answers Emmy's Round 1 (four of four addressed, [response](https://github.com/neomjs/neo-agent-brain/pull/1771#issuecomment-6902364526)). AC-1 is measured and ticked. CI on `ff723a27` was still running at handover.
- **Next:** once CI is green on `ff723a27`, re-request `@neo-gpt-emmy` on GitHub. The Fleet's review wake carries it, so no DM.
- **After merge:** a plane cut and an FM runtime update, since the receiver and the plane must both run the new code. Then AC-6, live: a sunset written behind a locked screen opens its fresh session within 60 s of the unlock, and the plane records `completed`. Post the receipt id and timings here.
- **Owner:** Ada, next session.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-15T23:09:12Z @neo-opus-ada referenced in commit `9bf3257` - "fix(wake): a held rotation waits for an opening, not a minute (#1770)

The 22:28Z sunset was lost on an unlocked screen: the receiver deferred
it 10 times on human-input-active (one pass every ~5.6 s) and expired it
60 s after its accept, like the 19:27Z and 19:43Z ones behind the lock.

Under the plane's signed hold, neither reason for that minute is left:
the plane holds the seat's wakes, and a session started by hand claims
the seat on its first prompt, which the holder fence reads before Cmd+N
(measured: the hand-started successor's first prompt replaced the
baseline holder at 22:46Z). So the hold alone is the deadline, and the
locked-time bookkeeping (lockedMs, lockedSince, rotationLockPatch) goes.
Lost activations, the one defer that takes the front from a person, are
capped at 5 (session-rotate-front-lost). Without a hold, 60 s stays."
- 2026-11-15T23:12:54Z @neo-opus-ada changed title from **A sunset during a locked screen never rotates, and nobody is told** to **A sunset rotation expires while the host is busy, and nobody is told**
- 2026-11-15T23:34:39Z @tobiu referenced in commit `71bebb8` - "fix(wake): a sunset rotation waits for the host's opening (#1770) (#1771)

* feat(wake): a locked screen pauses a rotation's age (#1770)

A locked screen keeps the front from every window, so a sunset written while the operator was away spent its 60 s on activations that could not succeed, then expired unseen. The receiver now reads the lock (CGSSessionScreenIsLocked, no privacy grant), defers screen-locked with its age paused, and resumes at the unlock behind the unchanged holder fence. The envelope carries the plane's holdUntil, so the receiver never types after the plane stops holding the seat's wakes, and the plane posts one defect note when it records a rotation incomplete.

* feat(wake): the incomplete note's hand-start prompt says so (#1770)

The note handed out the plane's own first prompt, which says the plane opened the session. A person who starts the seat from the note would type a false provenance, so the note now offers a prompt that says the session was started by hand after the rotation opened none.

* fix(wake): lock pause needs the hold; cutoff is absolute (#1770)

Round 1 of the review. Without a valid signed hold, a lock no longer extends the wall-clock minute, so an older plane keeps the 60 s it was promised. The fresh-session script compares an absolute cutoff, read after the holder check and right before Cmd+N, so a child that starts late opens nothing late. The adapter re-reads the lock at its last boundary and after a failed activation, and the receiver opens the paused interval for it. The incomplete note reports only the missing successor marker and asks for the seat's window to be checked before a hand start.

* fix(wake): the front is asserted again right before Cmd+N (#1770)

Round 1, RA-3's second half. A lock that engages after a successful activation took the front before Cmd+N with nothing to catch it. The script now asserts the front right before the cutoff check, and a loss there is retry-safe (isPreSessionFailure): only the tab shortcut, which opens no session, precedes it. The holder and cutoff refusals stay final.

* fix(wake): a held rotation waits for an opening, not a minute (#1770)

The 22:28Z sunset was lost on an unlocked screen: the receiver deferred
it 10 times on human-input-active (one pass every ~5.6 s) and expired it
60 s after its accept, like the 19:27Z and 19:43Z ones behind the lock.

Under the plane's signed hold, neither reason for that minute is left:
the plane holds the seat's wakes, and a session started by hand claims
the seat on its first prompt, which the holder fence reads before Cmd+N
(measured: the hand-started successor's first prompt replaced the
baseline holder at 22:46Z). So the hold alone is the deadline, and the
locked-time bookkeeping (lockedMs, lockedSince, rotationLockPatch) goes.
Lost activations, the one defer that takes the front from a person, are
capped at 5 (session-rotate-front-lost). Without a hold, 60 s stays."
- 2026-11-15T23:34:40Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-15T23:57:32Z

**Deployed.** The local plane was cut to `71bebb8c` (#1771's merge) at 23:56:18Z on 2026-11-15.
- All four services are healthy, with rev = neo-revision = `71bebb8c66d2`. Chroma and ingress were untouched, and the outage lasted 40 s.
- The host wake receiver (pid 43160) restarted at 23:56:12Z from the deploy home at `71bebb8c`, and its code carries the hold deadline and the front-lost cap.
- The orchestrator image emits `holdUntil`.

AC-6 is still open. It needs the next Claude-seat sunset taken while the host is busy (locked, or in use): its receiver record and the plane's `completed` line go here, with the timings.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-16T01:22:26Z @neo-opus-ada cross-referenced by #1777

