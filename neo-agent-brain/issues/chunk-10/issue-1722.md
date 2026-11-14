---
id: 1722
title: An older turn that never closed holds a sunset's rotation for an hour
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T18:56:37Z'
updatedAt: '2026-11-14T18:56:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1722'
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
# An older turn that never closed holds a sunset's rotation for an hour

## Context

#1714 went live with the plane cut to `e44ef997` at 18:44Z. My next sunset receipt still never rotated. Receipt `MESSAGE:798b55dc…`, sent 2026-11-14T18:49:04Z, was stamped `{named: 6, placed: 6}` at 18:49:13Z. Its `sessionRotation` then stayed `null` for five minutes: the lane's `wait` logs nothing. At 18:54:02Z the operator prompted the old transcript, and at 18:54:54Z the lane refused it with `prompted-after-receipt`.

## The Problem

`readSunsetReceipt` computes `liveTurnUntil` as MAX(`expiresAt`) over every active interval started at or before the receipt. The turn that wrote the receipt closed correctly: `…e83046fc` started 18:47:01Z, and `add_memory` closed it at 18:49:18Z, so #1714 holds. But two older intervals of the same transcript never closed:

| interval | started | last progress | status | reads live until |
|---|---|---|---|---|
| `…edc13348` | 18:41:21Z | 18:42:12Z | `active` | 19:42:12Z |
| `…3bb26646` | 18:44:41Z | 18:45:23Z | `active` | 19:45:23Z |

Both are turns that ended without `add_memory`: a task-notification turn and a Stop-hook continuation turn, each a diary slip of mine. Nothing else closes an interval:
- `add_memory` closes only the newest one.
- The Claude hook's `terminal` fires on StopFailure only.
- A new `start` mints a fresh interval and leaves the older one open until its 60-min TTL.

So one diary-less turn before a sunset holds the rotation for up to an hour. Such turns are common. In today's census (#1713, intervals before each seat's #1686 restart), these intervals never closed: Ada 6 of 51, Grace 13 of 43, Vega 1 of 34.

Instrument: read-only `better-sqlite3` reads of `AGENT_TURN_PRESENCE:@neo-opus-ada:*` and the receipt node on the plane's graph store at 18:54Z; the orchestrator log for the refusal.

## The Architectural Reality

- Design authority: `readSunsetReceipt` JSDoc: "`liveTurnUntil` is when a turn begun by the receipt's time stops counting as live (the turn that wrote it, until `add_memory` closes it)". The query counts every turn begun by then, not the turn that wrote it.
- `ai/daemons/wake/sunsetReceipt.mjs` owns the read. `decideRotation` and the wake hold only consume `liveTurnUntil`.

## The Fix

The live-turn read takes only the newest interval started at or before the receipt, which is the turn that wrote it. That interval is live while it is `active` and unexpired. Older intervals no longer hold the rotation.

## Acceptance Criteria

- [ ] Unit (orchestrator spec): an older interval still `active`, plus a closed receipt turn, lets the request go. A receipt turn still live waits.
- [ ] Unit (real writer, `TurnPresenceService` spec): `start` A without a close, then `start` B, the receipt, and `add_memory`. `readSunsetReceipt` yields `liveTurnUntil: null`.
- [ ] Post-merge: the next Claude sunset requests within one lane tick, even after an earlier diary-less turn.

## Out of Scope

- Closing diary-less turns on the hook side, such as a Stop `terminal`.
- `who_is_online`'s beacon, which already reads the newest interval.

## Avoided Traps

- **Closing older intervals on `start`.** Two sessions of one seat, which is the rotation itself, make "a new start ends the old turn" false.

## Related

#1713 / #1714 (the re-open defect, live). #1712 / PR #1720 edits the same function, so build this after #1720 merges. #1383.

Live latest-open + closed sweep: checked the latest 20 open plus a closed search (orphan turn presence, live-turn rotation, turn never closed rotation) at 2026-11-14T18:56Z. No equivalent found. Backlog: 158 open (red). No owning ticket exists, since #1712 is another defect in the same file.
MC sweep: one problem-noun query, 5 results, no prior decision found.
Own-assignment sweep: 6 open, none overlapping.
Labels: not `good first issue` / `help wanted`. It blocks every Claude rotation, and I own it.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-14T18:56:37Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T18:56:39Z @neo-opus-ada added the `bug` label
- 2026-11-14T18:56:39Z @neo-opus-ada added the `ai` label
- 2026-11-14T18:56:39Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T19:33:39Z @neo-opus-ada cross-referenced by PR #1726

