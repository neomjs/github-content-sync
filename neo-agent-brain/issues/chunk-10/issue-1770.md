---
id: 1770
title: 'A sunset during a locked screen never rotates, and nobody is told'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-15T20:54:48Z'
updatedAt: '2026-11-15T21:01:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1770'
author: neo-opus-ada
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
# A sunset during a locked screen never rotates, and nobody is told

## Context

The operator asked why `@neo-fable-mnemosyne`'s sunset at 19:43Z on 2026-11-15 opened no fresh session. Four sunsets ran that day:

| Receipt | Seat | Plane (`session-rotation`) | Host receiver record | Result |
|---|---|---|---|---|
| 17:35Z | `@neo-opus-ada` | requested (delivered) 17:35:34Z | typed | `completed` 17:37:39Z |
| 19:27Z | `@neo-opus-ada` | requested (delivered) 19:27:50Z | accepted 19:27:50.125Z · `deferCount` 5, `activation-lost` · `failed` `session-rotate-expired` 19:29:00.314Z | no session; the old one was resumed by hand at 20:07Z |
| 19:43Z | `@neo-fable-mnemosyne` | requested (delivered) 19:43:21Z | accepted 19:43:21.836Z · `deferCount` 5, `activation-lost` · `failed` `session-rotate-expired` 19:44:26.538Z | seat dark; last turn 19:42:22Z |
| 20:35Z | `@neo-opus-ada` | requested (delivered) 20:35:54Z | typed | `completed` 20:37:58Z |

**Observed.** Each of the ten failed attempts logged `Target app lost frontmost status after activation (-2700)`, then `rotation DEFERRED … (activation-lost)`. Each rotation ended with `rotation EXPIRED …: not typed within 60000 ms of its accept; failed without a keystroke`. The host's `loginwindow` log shows the screensaver starting at 18:32:26Z and locking the screen in the same millisecond (`startScreenLock: kLWLockFromScreenSaverIdleLaunch`; the 300 s it then reads is the keybag's grace period). There is no stop until 19:52:57Z. At that point it logged `saver is running, start the unlock`, `startUnlock: kLWLockFromScreenSaverIdleLaunch`, and `Keybag is locked, setting promptForPassword = YES`. So the screen was locked from 18:32:26Z to 19:52:57Z, and both lost rotations fell inside that span.

**Inferred.** The idle clock (`HIDIdleTime`) reads a locked screen as quiet, so each attempt activates the seat's window. The lock screen keeps the front, and the rotation spends its 60 s on attempts that cannot succeed.

## The Problem

A sunset written while the operator is away is lost whenever the screen locks first, and the loss is silent.

1. The receiver gives a rotation 60 s from its accept, and a lock lasts longer than that.
2. The expiry stays on the host. The plane cannot read the receiver's records: its container declares no `fleet.wakeReceiverRecordsDir`, and #1722 left that back-channel out of scope. So the receipt stays `requested` and the plane holds the seat's wakes for `sessionRotation.windowMs` (3 h). The plane then records `incomplete` as one ERROR log line. Until someone notices, the seat is dark and its wakes are held. At 20:5xZ Mnemosyne's receipt was still open.

## The Architectural Reality

- `ai/daemons/wake/receiver.mjs:471` fails a rotation when `isRotationExpired(acceptedAt, now)` (`sessionRotation.mjs:200`) holds, against `acceptedAt + ROTATION_MAX_AGE_MS` (60 s, `sessionRotation.mjs:61`). The holder fence (`receiver.mjs:483` on, re-read by the script right before Cmd+N) fails a rotation once another session claims the seat.
- `localWakeAdapters.mjs` defers `activation-lost` at `:783` and gates typing on `readHumanIdleMs` at `:1265` (`ioreg`, `HIDIdleTime`). Nothing reads the lock state.
- `decideRotation` (`sessionRotation.mjs:354`) answers `wait (receiver-has-request)` for a delivered rotation until `windowMs`, then `incomplete`. `runSessionRotation` (`orchestrator/scheduling/sessionRotation.mjs:87`) only logs it. The plane already publishes defect notes through `Orchestrator#publishDefectNote` (`Orchestrator.mjs:399`), and `DiskPressureDefectService` uses it.
- The console session's lock flag is readable without a privacy grant (`CGSessionCopyCurrentDictionary`, also `ioreg -n Root -d1` → `IOConsoleUsers`). `CGSSessionScreenIsLocked` is absent while unlocked (measured 20:4xZ). Its presence while locked is documented behaviour that has not yet been measured on this host.

Design authority: `ai/daemons/wake/sessionRotation.mjs:12-18`, "It waits for that pause only {@link ROTATION_MAX_AGE_MS}: a sunset is terminal and its fresh session starts at once, while every minute of waiting lets wakes reach a session that has ended and lets a person restart the seat by hand, after which a late rotation is a second live session of the seat." Neither reason holds while the screen is locked. The plane holds the seat's wakes, so none reach the ended session. Nobody at the host can restart the seat, and a restart from elsewhere claims the seat on its first prompt, which the holder fence reads. The bound measures time spent competing with a person for the front, and that is what this ticket keeps.

## The Fix (one PR)

1. **Receiver: locked time does not age a rotation.** Add `readScreenLocked()` beside `readHumanIdleMs` in `localWakeAdapters.mjs`. A rotation that finds the screen locked defers with a new reason `screen-locked`, with no activation and no keystroke. Its 60 s counts only unlocked time. An unreadable lock state keeps today's path, because the frontmost assertion still guards every keystroke. The 3 s quiet window and the holder fence are unchanged. On this host the screensaver locks the screen as it starts, so the lock flag covers the screensaver too.
2. **Envelope: the receiver never types after the plane has given up.** `buildRotationEnvelope` carries `payload.holdUntil`, the receipt's `sentAt` plus `windowMs`. A `screen-locked` wait still open at `holdUntil` fails `session-rotate-expired`. An envelope without the field keeps the 60 s bound, which covers an older plane.
3. **Plane: `incomplete` is never only a log line.** `runSessionRotation` publishes one `defect-note:` per receipt through `publishDefectNote` when it records `incomplete`, naming the seat, the receipt and the sunset time.
4. **Contract docs.** The `sessionRotation.mjs` header and the `windowMs` JSDoc in `ai/configBase.mjs` (text only; read ADR-0019 before the edit) say the age runs only while the screen is unlocked.

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `session/rotate` `payload.holdUntil` | `buildRotationEnvelope` | ISO time the plane's hold on the receipt ends | absent → 60 s from accept | module header | unit |
| defer reason `screen-locked` | `sessionRotation.mjs` constants | rotation waits, age paused, nothing typed | lock unreadable → today's path | constant JSDoc | unit + one live lock |
| `incomplete` defect note | `runSessionRotation` | one `defect-note:` per receipt | publisher throws → logged, record still written | lane JSDoc | unit |

Decision Record impact: none. No ADR governs rotation. This amends the receiver contract #1722 set in the `sessionRotation.mjs` header.

## Acceptance Criteria

- [ ] AC-1: On the operator host, `readScreenLocked()` returns `true` during a real lock and `false` unlocked, each measured once by a sampler that only reads. Receipts go in the PR.
- [ ] AC-2 (unit): a rotation accepted while locked defers `screen-locked` past 60 s with no activation and no keystroke. After the unlock it types at the first 3 s pause within 60 s of unlocked time.
- [ ] AC-3 (unit): a seat that another session claims during the lock fails the rotation `session-rotate-superseded` after the unlock, without a keystroke.
- [ ] AC-4 (unit): a `screen-locked` wait still open at `holdUntil` fails `session-rotate-expired` without a keystroke. Without `holdUntil`, the bound stays 60 s from `acceptedAt`.
- [ ] AC-5 (unit): `runSessionRotation` publishes exactly one `defect-note:` when it records `incomplete`. It publishes none on later passes and none for `completed` or `refused`.
- [ ] AC-6 (post-merge, live, after the plane cut and the FM runtime update): a sunset written while the screen is locked opens its fresh session within 60 s of the unlock, and the plane records `completed`. A comment cites the receipt id and the timings.

## Out of Scope

- A receiver → plane back-channel for other failures. The plane still learns of those only at `windowMs`, now with a note.
- The values of `windowMs` and `ROTATION_MAX_AGE_MS`.
- The host's lock and screensaver settings.
- Wake delivery during a lock.

## Avoided Traps

- **#1722's "longer windows or more retries".** That trap's reason is wakes reaching an ended session. During the lock the plane holds them, and `holdUntil` ends the wait when the hold ends.
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

Retrieval Hint: "rotation expired activation-lost locked screen screensaver" · "session-rotate-expired no fresh session after sunset"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-15T20:54:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-15T20:54:50Z @neo-opus-ada added the `bug` label
- 2026-11-15T20:54:50Z @neo-opus-ada added the `ai` label
- 2026-11-15T20:54:51Z @neo-opus-ada added the `agent-os` label
- 2026-11-15T21:21:00Z @neo-opus-ada cross-referenced by PR #1771
- 2026-11-15T21:34:35Z @neo-opus-ada referenced in commit `44d3380` - "feat(wake): the incomplete note's hand-start prompt says so (#1770)

The note handed out the plane's own first prompt, which says the plane opened the session. A person who starts the seat from the note would type a false provenance, so the note now offers a prompt that says the session was started by hand after the rotation opened none."

