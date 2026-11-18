---
id: 1841
title: Async PostToolUse turn progress holds Claude sunset rotation 60 min
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-18T15:37:35Z'
updatedAt: '2026-11-18T19:16:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1841'
author: neo-gemini-julia
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
closedAt: '2026-11-18T19:16:57Z'
---
# Async PostToolUse turn progress holds Claude sunset rotation 60 min

## Context

On `2026-11-18` (deployed Brain revision `d172de48`), multiple Claude Code seats failed to start fresh sessions after `/session-sunset`, sitting silent in `waiting (live-turn)` until manual intervention triggered `rotation refused · prompted-after-receipt`.

Read-only SQLite measurements on the live graph store (`@neo-opus-ada` and `@neo-fable-mnemosyne`):

| Seat / Event | Node ID | Timestamps & Fields |
|---|---|---|
| Ada handover 1 | `MESSAGE:de397fdd-ed56-421c-8624-60af430b95e1` | `sentAt: 14:36:54.131Z`, `writerKey: v1:307def5255aca2a8e72d087143a8b2f1`, `writerTurn: b4622140` |
| Ada sub-turn N presence | `AGENT_TURN_PRESENCE:@neo-opus-ada:b4622140-72c6-4566-acb1-aa1bc5e83fdc` | `startedAt: 14:34:57.966Z`, `lastProgressAt: 14:36:55.751Z`, `status: active`, `expiresAt: 15:36:55.751Z`, `source: claude-post-tool-use` (`add_message`) |
| Ada sub-turn N+1 start | `AGENT_TURN_PRESENCE:@neo-opus-ada:1267b06e-1b8d-4ab9-9ec9-c56d2cddf736` | `startedAt: 14:36:55.749Z`, `lastProgressAt: 14:37:26.303Z`, `status: terminal` (`add_memory` `8c71d21a`) |
| Ada addendum handover 2 | `MESSAGE:4d664b7c-4b10-4cea-bd66-a316df2cd06e` | `sentAt: 14:37:18.163Z`, `inReplyTo: MESSAGE:de397fdd`, same `writerKey`, `writerTurn: 1267b06e`; orchestrator `waiting (live-turn)` at `14:38:18Z`, refused `prompted-after-receipt` at `14:50:53Z` |
| Mnemosyne sub-turn overlap (4×) | `AGENT_TURN_PRESENCE:@neo-fable-mnemosyne:*` | `14:27:37.568Z >= .551Z` (`+17 ms`), `14:49:10.126Z >= .123Z` (`+3 ms`), `15:14:53.048Z >= .046Z` (`+2 ms`); plus `MESSAGE:33b37adb` refused `prompted-after-receipt` at `13:55:39Z` |

Notice that sub-turn `b4622140`'s `PostToolUse` write settled **2 ms after** sub-turn `1267b06e` started (`14:36:55.751Z >= 14:36:55.749Z`).

## The Problem

#1777 introduced `releaseByWriterStamp` so an earlier unclosed `active` interval (60-minute TTL) would not hold a sunset rotation, explicitly relying on sequential turns inside one session satisfying `previous.lastProgressAt < next.startedAt`:

In Claude Code, `PostToolUse` (`progress`) is registered `async: true` in `ai/scripts/lifecycle/hooks/claude/turnPresenceHook.mjs`, whereas `UserPromptSubmit` (`start`) runs synchronously. Whenever sub-turn `N` calls `add_message` without calling `add_memory` and sub-turn `N+1` starts immediately afterward:

1. The background HTTP `PostToolUse` write for `N` routinely lands **2–17 ms (or more under `mc-server` CPU throttling) after** `N+1.startedAt` (`ownStart`), leaving `N` in `status: 'active'` with a 60-minute `expiresAt` and `lastProgressAt >= ownStart`.
2. Because `AGENT_TURN_PRESENCE` nodes carry no `writerKey`, `releaseByWriterStamp` (`ai/daemons/wake/sunsetReceipt.mjs`) evaluates `row.status === 'active' && row.expiresAt > nowIso && row.lastProgressAt >= ownStart` as true, misclassifies the writer's own predecessor sub-turn `N` as a concurrent second chat, and blocks `session/rotate` for up to 60 minutes until manual operator prompting sets `promptedTurnAt` (`rotation refused · prompted-after-receipt`).
3. Additionally, a second handover (addendum) sent by the same `writerKey` before the 60s orchestrator pass stamps `requestedAt` (`isOpenAt(null, at)` evaluates `false` in `readSunsetReceipt`) starts a brand-new receipt chain instead of folding into the first handover.

## The Architectural Reality

- `ai/daemons/wake/sunsetReceipt.mjs` (`readSunsetReceipt`, `releaseByWriterStamp`): holds `liveTurnUntil` for `others.filter(row => row.status === 'active' && row.expiresAt > nowIso && row.lastProgressAt >= ownStart)` without distinguishing same-session sequential sub-turns; folds amendments in `readSunsetReceipt` only when `isOpenAt(receipt.properties.sessionRotation, at)` is already true (`requestedAt` stamped).
- `ai/scripts/lifecycle/hooks/claude/turnPresenceHook.mjs` (`sessionTurnStore`, `ASYNC_ACTIONS`, `recordClaudeTurnPresence`): `UserPromptSubmit` is synchronous (`start`) and overwrites `sessionTurnStore` for `session_id` without closing/superseding the previous turn id or attaching `writerKey`; `PostToolUse` is async (`ASYNC_ACTIONS`).
- `ai/scripts/lifecycle/hooks/claude/writerStampHook.mjs`: already computes deterministic `writerKey` (`v1:...` SHA-256 of seat + `session_id`) on `add_message` / `add_memory`.
- `ai/mcp/server/memory-core/helpers/TurnPresenceHookWriter.mjs`, `ai/services/memory-core/TurnPresenceService.mjs`, and `ai/mcp/server/memory-core/openapi.yaml`: persist `AGENT_TURN_PRESENCE` nodes without `writerKey`.

Design authority: #1777 ("Liveness after the release. Another open, unexpired, non-wake interval whose last progress is at or after the writer's sunset-turn start ran concurrently with the sunset (a session's turns are sequential), so it holds the release as live-turn. One whose last progress predates that turn is history and does not hold.").

## The Fix

1. **Stamp `writerKey` on `AGENT_TURN_PRESENCE`:** Add optional `writerKey` (string) to `record_turn_presence` (`ai/mcp/server/memory-core/openapi.yaml` + `TurnPresenceService.mjs`), passed from `turnPresenceHook.mjs` → `TurnPresenceHookWriter.mjs` using the same seat + `session_id` digest helper as `writerStampHook.mjs` (never logging clear session ids or keys). When `recordClaudeTurnPresence` runs `action === 'start'` and `sessionTurnStore.read(sessionId)` holds an earlier `previousTurnId` for the same `session_id`, optionally close/supersede that predecessor interval cleanly.
2. **Exclude same-`writerKey` predecessor sub-turns from `liveTurnUntil`:** In `releaseByWriterStamp` (`ai/daemons/wake/sunsetReceipt.mjs`), exclude intervals with `row.writerKey === key && row.startedAt <= ownStart` from `liveTurnUntil` (those are earlier sub-turns of the same sequential session whose async `PostToolUse` settled after `ownStart`, never a concurrent second chat; intervals with a different or missing `writerKey` and `lastProgressAt >= ownStart` still hold `live-turn`).
3. **Fold pre-`requestedAt` same-`writerKey` handover addenda:** In `readSunsetReceipt`, when consecutive `sunset-protocol-handover` self-messages carry the same non-empty `writerKey` (and no non-handover self-message or successor marker intervened) before `requestedAt` is stamped, fold the addendum into the receipt chain so an addendum sent seconds after the handover does not orphan the prior receipt or stall waiting for a separate close.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `record_turn_presence` (`ai/mcp/server/memory-core/openapi.yaml`, `TurnPresenceService.mjs`) | #1777 writer-stamp pattern | Accept optional `writerKey` string on `start`/`progress`/`terminal` and persist on `AGENT_TURN_PRESENCE` node | Absent `writerKey` leaves node unstamped; existing behavior unchanged | JSDoc on `TurnPresenceService#recordTurnPresence` | Unit test + OpenAPI schema parity |
| `releaseByWriterStamp` & `readSunsetReceipt` (`ai/daemons/wake/sunsetReceipt.mjs`) | #1777 + #1739 | Exclude `row.writerKey === key && row.startedAt <= ownStart` from `liveTurnUntil`; fold pre-`requestedAt` same-`writerKey` handover addenda | Missing `writerKey` keeps existing `lastProgressAt >= ownStart` guard and existing amendment rule | Module JSDoc in `sunsetReceipt.mjs` | Unit specs in `sessionRotation.spec.mjs` |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `record_turn_presence` OpenAPI schema, `TurnPresenceService.mjs`, `TurnPresenceHookWriter.mjs`, and `turnPresenceHook.mjs` stamp optional `writerKey` on `AGENT_TURN_PRESENCE` nodes without exposing `writerKey` in logs or error strings.
- [ ] Unit test in `test/playwright/unit/ai/daemons/orchestrator/scheduling/sessionRotation.spec.mjs` reproduces the `2 ms` late async `PostToolUse` overlap (`b4622140.lastProgressAt = ownStart + 2 ms` on `writerKey === key` and `startedAt < ownStart`) and asserts `releaseByWriterStamp` returns `liveTurnUntil: null` so `decideRotation` returns `request`.
- [ ] Positive control unit test verifies an `active` interval with a *different* or absent `writerKey` and `lastProgressAt >= ownStart` still holds `live-turn`.
- [ ] Unit test verifies a pre-`requestedAt` same-`writerKey` handover addendum (`inReplyTo` or consecutive self-handover) folds into the existing receipt chain instead of leaving an unhandled prior handover.
- [ ] `[L4-deferred — operator handoff needed]` On the deployed plane after rebuild, a multi-turn Claude `/session-sunset` (including an immediate addendum or tool call before handover) rotates cleanly into a fresh session without holding 60 min in `live-turn`.

## Out of Scope

- Host wake receiver `primary-session` CPU spin blocking wake/rotation responsiveness (#1840).
- Locked-screen receiver expiry behavior (#1770).
- Roster card `"to unknown"` formatting when `newSessionId` is `null` (separate `neomjs/neo-agent-institution` ticket).

## Avoided Traps

- **Adding a constant clock tolerance (`ownStart + N ms`) in `releaseByWriterStamp`:** arbitrary under `mc-server` CPU throttling where async `PostToolUse` HTTP writes can lag hundreds of milliseconds.
- **Inheriting `writerKey` onto unstamped `add_memory` terminals:** already falsified in #1777 (D#19401 Option 1).

## Related

#1777, #1770, #1739, #1722, #1840, neomjs/neo Discussion #19401

---

Live latest-open + closed sweep: checked latest 20 open of 154 (`red` band) + open/closed searches (`sessionRotation`, `prompted-after-receipt`, `sunsetReceipt`, #1777, #1770, #1739) at 2026-11-18T15:36:25Z; Out of Scope / Avoided Traps read on #1777 and #1770; no open owner for rotation race.
A2A in-flight sweep: latest 30 messages read across all read states (`2026-11-18T14:37Z–15:35Z`); no peer claim on `sunsetReceipt.mjs` / `turnPresenceHook.mjs`.
MC sweep: `"rotation refused prompted-after-receipt claude session sunset fresh session"`, 8 results (#1722/#1777 history only, no prior session covering the async `PostToolUse` race).
Own-assignment sweep: 1 open (`#1230`), non-overlapping.
Structure map (`node ai/scripts/diagnostics/structureMap.mjs --files --loc`, exit 0): owning folders `ai/daemons/wake`, `ai/scripts/lifecycle/hooks/claude`, `ai/services/memory-core`; no new files required.

unowned-rationale: Ready for a maintainer or contributor; requires live container rebuild/restart for the `[L3, post-merge]` AC, so `good first issue` / `help wanted` labels are omitted per ticket-create rules.

Origin Session ID: 613064f8-74aa-4baa-8ee7-6a0b8540f31b
Retrieval Hint: "async PostToolUse lastProgressAt ownStart live-turn releaseByWriterStamp claude sunset rotation prompted-after-receipt"

## Timeline

- 2026-11-18T15:38:05Z @neo-gemini-julia cross-referenced by #1842
- 2026-11-18T15:38:07Z @neo-gemini-julia cross-referenced by #1023
- 2026-11-18T15:38:21Z @neo-gemini-julia added the `bug` label
- 2026-11-18T15:38:21Z @neo-gemini-julia added the `ai` label
- 2026-11-18T15:38:22Z @neo-gemini-julia added the `architecture` label
- 2026-11-18T15:38:22Z @neo-gemini-julia added the `agent-os` label
- 2026-11-18T15:38:31Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-18T16:24:52Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-18T16:47:20Z @neo-gemini-julia cross-referenced by PR #1844
- 2026-11-18T17:56:46Z @neo-gemini-julia cross-referenced by PR #1845
- 2026-11-18T18:42:45Z @neo-gemini-julia referenced in commit `137d727` - "fix(wake): unify turn-presence writerKey identity source and sync sunset docs (#1841)"
- 2026-11-18T19:16:56Z @tobiu referenced in commit `0b0cc33` - "fix(wake): ignore predecessor PostToolUse progress in sunset rotation (#1841) (#1844)

* fix(wake): ignore predecessor PostToolUse progress in sunset rotation (#1841)

* fix(wake): unify turn-presence writerKey identity source and sync sunset docs (#1841)"
- 2026-11-18T19:16:58Z @tobiu closed this issue

