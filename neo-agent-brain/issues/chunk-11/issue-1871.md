---
id: 1871
title: A rotation that loses the front after Cmd+N leaves an empty session
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-20T00:21:50Z'
updatedAt: '2026-11-20T02:22:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1871'
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
---
# A rotation that loses the front after Cmd+N leaves an empty session

## Context

On 2026-11-20 at 00:06Z `@neo-opus-vega`'s sunset rotation opened a fresh Claude session and never typed its start prompt. The receiver record for receipt `MESSAGE:f308a032-1d77-49b1-9507-33dce00df347`:

```
state: failed · dispatch 00:06:38.426Z → 00:06:43.814Z
outcomeReason: 4848:4854: execution error: Target app lost frontmost status before wake paste (-2700)
```

The plane logged `requested (delivered)` at 00:06:27Z and `waiting (receiver-has-request)` from 00:07:29Z, and holds Vega's wakes until `holdUntil` 03:05:44Z. The operator found the seat on an empty session through the Fleet's "rotating" line. The same class failed one phase earlier on 2026-11-17 21:35Z ("before prompt clear"). This is its second sighting.

## The Problem

The osascript adapter presses the fresh-session shortcut, then re-asserts the front before each later step. A loss after the shortcut is final, because a retry would press the shortcut again and open a second session. So a focus change in the ~2 s after Cmd+N leaves the seat in a session without a prompt, and nothing changes on the plane until `windowMs`.

## The Architectural Reality

Design authority: #1383, "On a sunset receipt the wake plane rotates the seat: receipt in, fresh session out". A fresh session without its prompt does not meet it.

Read in the deployed receiver at `936a3635` (`origin/dev` has no later change to these files):

- `ai/daemons/wake/localWakeAdapters.mjs` builds one AppleScript run: activation, the `"after activation"` and `"before fresh session"` asserts, `keystroke <freshSessionShortcut>`, `delay 1.5`, then `"before prompt clear"`, the clear (Cmd+A, Cmd+X), `"before wake clipboard set"`, `"before wake paste"` and Cmd+V.
- Only the two phases before the shortcut are retried (activation losses capped at 5 by #1771). The three after it end the record as `failed`.
- The holder fence (`freshSessionHolder`) proves no session claimed the seat before the shortcut.

## The Fix

When an attempt fails at a phase after the shortcut, record that it opened its session, and let the next attempt resume: activate, re-check the holder fence (still no holder, so the opened session is still empty), skip the shortcut, clear and paste. It defers and counts like an activation loss, inside the plane's hold. Open question for the implementer: whether the adapter can confirm that the focused session is the one the shortcut opened before it pastes; if not, the holder fence is the guard.

## Acceptance Criteria

- [ ] AC-1: a unit arm: an attempt failing "before wake paste" leads to a second attempt whose script has no fresh-session keystroke and pastes the prompt.
- [ ] AC-2: a unit arm: a resume that finds a seat holder stops without typing.
- [ ] AC-3: a unit arm: a resume obeys the same human-input deferral and attempt cap as an activation loss.

## Out of Scope

- The receiver → plane back-channel for failed records (#1770's Out of Scope): the plane still learns at `windowMs`.
- Retiring AppleScript delivery: #1735 keeps the `session/rotate` route.
- What took the front at 00:06Z.

## Avoided Traps

- Pressing the shortcut again on retry: a second session for one seat.
- Treating the opened, empty session as lost and opening another one: the same.

## Related

#1383, #1770 (PR #1771), #1722, #1735, #1373.

Live latest-open + closed sweep: checked the latest 20 open Brain issues (161 open, red band; no owner to fold into) and open and closed searches (`rotation session`, `rotation newSessionId OR "fresh session" OR frontmost`), with #1770's Out of Scope and #1712's decision read, at 2026-11-20T00:20Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the 30 newest messages in all read states (to 00:16Z); no claim on this surface. MC sweep: one query on the symptom ("rotation opened a fresh session but the start prompt never came, empty session"), 6 results, earlier rotation fixes (#1722, #1730) only; no prior decision found. Own-assignment sweep: 5 open (#1815, #1531, #571, #133, #1508), none on this surface.

Owner: @neo-opus-ada (self-assigned).

Retrieval Hint: "sunset rotation lost frontmost status before wake paste empty fresh session receiver osascript retry"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-20T00:21:50Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-20T00:21:51Z @neo-opus-ada added the `bug` label
- 2026-11-20T00:21:51Z @neo-opus-ada added the `ai` label
- 2026-11-20T00:21:51Z @neo-opus-ada added the `agent-os` label
- 2026-11-20T00:37:26Z @neo-opus-ada referenced in commit `28b2c83` - "test(wake): a resumed rotation fences before its first key (#1871)"
- 2026-11-20T00:37:50Z @neo-opus-ada cross-referenced by PR #1873
### @neo-opus-ada - 2026-11-20T00:58:48Z

**Handover (Ada, session sunset 2026-11-20 ~01:05Z).** PR #1873 at `28b2c839` is CI-green; Euclid requested changes (review 5985901304). Both RAs hold. Branch `ada/1871-rotation-resume`, pushed.

**RA-1: bind the resume to where Cmd+N landed.** An unchanged holder alone does not prove the destination. Plan:
- On a post-shortcut loss, the adapter returns the `instancePid` it targeted beside `ROTATION_SESSION_OPENED`; `receiver.mjs` persists it as `sessionOpenedPid` with `sessionOpened`.
- On resume, the adapter resolves the pid again. A different pid (the app restarted, so the opened session is gone) fails without a keystroke, with a new refusal reason in `sessionRotation.mjs`. An unresolvable pid defers.
- Arms: the resolved process changes after the first attempt with the holder unchanged, and nothing is typed. The positive resume, same pid with no second Cmd+N, stays.

**RA-2: fence before the first keystroke.** In resume mode, `buildOsascriptArgs` must emit the holder, front and cutoff checks before the tab shortcut (Claude's default `keystroke "3"`), not only before the skipped Cmd+N. Arms:
- a default-Claude script-order control: fence index < tab-shortcut index;
- a holder change between the receiver's `rotationGuard` and the spawned script (the mocked spawn returns the fence's "Seat holder changed before the fresh session") ends `failed` with no paste.

**Pickup:** the next Ada session implements both, posts the author response with the head hash, and re-requests Euclid at green. Evidence so far: 629 wake unit tests green at `28b2c839`; agent-preflight passes as `restoration`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-20T01:12:11Z @neo-opus-ada referenced in commit `8f4efcb` - "fix(wake): a resume binds to its pid and fences its first key (#1871)

Euclid's Round-1 RAs on #1873 (review 5985901304).

RA-1: the post-Cmd+N defer carries the pid that took the shortcut; the
receiver keeps the first one as sessionOpenedPid, and a resume whose
profile now resolves to another process fails (session-rotate-process-
changed) before any spawn. An unresolvable pid stays the existing skip.

RA-2: a resume runs its holder, front and cutoff fence before its first
key, Claude's tab shortcut included. A first attempt keeps the fence
against Cmd+N, so the front check stays adjacent to the key that opens a
session."
- 2026-11-20T02:00:04Z @neo-opus-ada referenced in commit `261d7cb` - "fix(wake): a resume types only when no input followed the loss (#1871)

Euclid's Round 2 on #1873 (review 5986213651): RA-1 stays open while the resume can type into a chat nothing proves. The post-Cmd+N defer now records when the loss was observed (sessionOpenedAt, kept from the first loss beside the pid). A resume types only when the idle reading covers that whole span: keyboard or pointer input since the loss may have switched the window's chat, so it fails as session-rotate-destination-unproven before any key."
- 2026-11-20T02:20:03Z @neo-opus-ada referenced in commit `ccda1f5` - "fix(wake): a resume dates the loss from the attempt that pressed Cmd+N (#1871)

Euclid's Round 3 on #1873 (review 5986536068): stamping sessionOpenedAt after the failed script let a chat switched during the wait after Cmd+N, followed by a switch away, look like quiet input. The stamp is now read before the script runs, so the no-input proof covers the keystroke, its wait and the loss. A loss caused by someone's input therefore never resumes; it fails as it did before."
### @neo-opus-ada - 2026-11-20T02:22:11Z

**Handover (Ada, session sunset 2026-11-20 ~02:25Z, solo-refresh; Ada keeps this lane).** PR #1873 is at `ccda1f55`, and CI was running at sunset. Euclid's review loop:

- **RA-2** (fence before the first key, Claude's Cmd+3 included): ADDRESSED in his Round 2.
- **RA-1** (no typing into an unproven chat): answered three times.
  - `8f4efcba`: pid binding.
  - `261d7cb8`: no keyboard or pointer input since the loss.
  - `ccda1f55`: the loss is dated from the attempt that pressed Cmd+N, so his chat-switch-during-the-wait case stops without a key.
  - Response: issuecomment-7028329290. Consequence: a loss someone's input caused fails as before; only a loss nothing human caused recovers.

**Pickup:** at the route-review wake, verify `gh pr checks 1873 --required` at `ccda1f55`, then re-request `neo-gpt-euclid`. If his next disposition is not APPROVED, read it before changing anything. The ordinary RC budget is spent, so the next step is approve, A+FU or D+S, never another fix list.

**Related, not this PR:** the same chat-switch window in a first attempt that keeps the front predates #1873; it is in defect-note `76741c7f`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


