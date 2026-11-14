---
id: 1713
title: A tool call after add_memory re-opens the turn it closed
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T17:33:00Z'
updatedAt: '2026-11-14T18:46:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1713'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-14T18:01:09Z'
---
# A tool call after add_memory re-opens the turn it closed

## Context

A sunsetting Claude seat should continue in a fresh session (#1383; the receipt reader was fixed in #1710 / #1711, plane cut to `f9b0044b` at 17:16Z). My sunset receipt at 2026-11-14T17:18:57Z was the first stamped one that could rotate end to end. It did not. The rotation lane waited silently on `live-turn` until the operator started the successor session by hand at 17:21:44Z, then recorded `refused (prompted-after-receipt)` at 17:22:34Z.

## The Problem

`readSunsetReceipt` holds the rotation while any of the seat's turn intervals that began by the receipt's time is `active` and unexpired. Since #1686, a Claude turn interval never ends `terminal`:

1. `add_memory` closes the newest active interval server-side (`MemoryService` step 6 → `recordTurnPresence({action: 'terminal'})`, no `turnId`).
2. The Claude PostToolUse hook then fires for that same `add_memory` call and for every later tool call of the turn. Since #1686 it sends the `turnId` its `start` stored (`turnStore.read(sessionId)` in `ai/scripts/lifecycle/hooks/claude/turnPresenceHook.mjs`). Before #1686 the hook sent no `turnId` on progress, so this progress found no open interval and was the documented no-op.
3. `recordTurnPresence` with an explicit `turnId` skips the open-interval lookup. It writes `status: 'active'`, `terminalState: null` and `expiresAt: now + ttlMs` (60 min) over the closed row.

Specimen: my sunset turn's interval (`…3cf11a22`, started 17:16:28Z). Its last write is `claude PostToolUse mcp__ccd_turn__wrap_up` at 17:19:07.707Z, 2.3 s after the turn's `add_memory` (17:19:05.410Z). It reads `active` until 18:19:07Z, so without the manual start the rotation would have waited until then.

| seat | before the seat's first start on the runtime carrying #1686 (terminal · active) | after (terminal · active) |
|---|---|---|
| Ada (00–12Z / 13–17Z) | 45 · 6 | 0 · 29 |
| Grace (00–12Z / 14–17Z) | 30 · 13 | 0 · 16 |
| Vega (00–11Z / 14–17Z) | 33 · 1 | 0 · 14 |

Instrument: read-only `better-sqlite3` query on the container plane's graph store (`Nodes` id range `AGENT_TURN_PRESENCE:@neo-opus…`, `startedAt ≥ 2026-11-14T00:00Z`, bucketed by start hour), plane at Brain `f9b0044b`, 2026-11-14T17:29Z. Positive control: the same query returns each seat's `terminal` rows before its restart. The `active` rows before the restart are turns that never called `add_memory`. They are out of scope here.

Receipts today: 8 sunset receipts, 0 rotation requests. Only mine (17:18:57Z) was stamped and unrefused long enough to isolate this guard. The earlier ones were blocked first by #1710.

## The Architectural Reality

- Design authority: `recordTurnPresence` JSDoc: "A `progress` or `terminal` event with no open interval is a **no-op, not an error**: the turn it would refresh has already expired or closed". And `readSunsetReceipt` JSDoc: "`liveTurnUntil` is when a turn begun by the receipt's time stops counting as live (the turn that wrote it, until `add_memory` closes it)".
- `targetTurnId = turnId || …` trusts an explicit id without reading the row's `status` (`ai/services/memory-core/TurnPresenceService.mjs`).
- Readers of an `active` interval:
  - the rotation's live-turn guard (`ai/daemons/wake/sunsetReceipt.mjs`). This is the observed failure.
  - `getFreshTurnPresence` → `who_is_online`'s mid-turn rescue. An idle Claude seat reads online for `freshMs` (30 min) after its last tool call.
  - `_withPullVerdicts` → `projectPullRouteVerdict`, which uses the turn as its long-turn discriminator (#1681 Avoided Traps).
- Placement: `npm run ai:structure-map -- --files --loc` ran (exit 0). The fix stays in `ai/services/memory-core/`, with no new file.
- Decision Record impact: none.

## The Fix

In `recordTurnPresence`, an explicit `turnId` never re-opens a closed interval. When its row is `terminal`, `progress` and `terminal` return the existing `noop` / `no-active-turn` shape and write nothing. A row that is past `expiresAt` but was never closed stays joinable by its own turn: that progress means a tool call outlived the TTL and the turn is alive, and every reader already bounds liveness by `expiresAt`. `start` is unchanged, since it mints its id. The hook is unchanged: its stored id goes stale harmlessly until the next `start` overwrites it.

## Acceptance Criteria

- [ ] Unit: `start` → terminal without `turnId` (the `add_memory` path) → `progress` with the explicit `turnId` returns `noop`. The row stays `terminal` / `completed`.
- [ ] Unit: a `terminal` with the explicit `turnId` after that close keeps `terminalState: 'completed'`.
- [ ] Unit: `progress` with an explicit `turnId` on an open interval still refreshes it, and past `expiresAt` it still joins its own turn (existing behavior, pinned).
- [ ] Unit: with the row closed and a later explicit-`turnId` progress, `readSunsetReceipt` yields `liveTurnUntil: null`.
- [ ] Post-merge (after the plane cut and a seat restart): the census query shows Claude intervals ending `terminal` again. The next Claude sunset receipt leaves `live-turn` within one lane tick (60 s) of its stamp.

## Out of Scope

- #1712 (the successor marker behind `newer-session` / `complete`).
- Intervals of turns that never call `add_memory`. They expire on the TTL.
- The Codex and Kimi hooks: their progress path is not the one measured here.

## Avoided Traps

- **Clearing the hook's turn store on `add_memory`'s PostToolUse.** That fix is hook-local and Claude-only, and any other writer with an explicit id keeps the hole. The invariant belongs in the service.
- **Shortening `ttlMs`.** It shrinks the window and keeps the wrong state.
- **Dropping the rotation's `live-turn` guard.** The guard is right; its input is wrong.
- **Refusing a named progress past `expiresAt`.** It would make a live turn whose tool call outlived the TTL read dead. Expiry is the reader's bound, not the writer's.

## Related

- Regression from #1686 (resolves #1681), my PR.
- #1383 (rotation), #1710 / #1711 (receipt reader), #1712 (successor marker).

Live latest-open + closed sweep: checked the latest 20 open plus a closed search (turn presence, add_memory terminal, live turn rotation, presence interval active) at 2026-11-14T17:31Z, and read the Out of Scope / Avoided Traps of #1681. No equivalent or prior exclusion found. Backlog: 154 open (red). No owning ticket exists, since #1712 is a different defect in a different file.
MC sweep: "seat still reads mid-turn after its turn ended; turn presence interval stays active after add_memory; rotation waits on live turn", 6 results, no prior decision found.
Own-assignment sweep: 7 open, none overlapping.
Labels: not `good first issue` / `help wanted`. It blocks every Claude rotation, and it is a regression from my own #1686 that I own.

Retrieval Hint: "turn presence interval re-opened after add_memory", "rotation waits on live-turn".

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-14T17:33:00Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T17:33:02Z @neo-opus-ada added the `bug` label
- 2026-11-14T17:33:02Z @neo-opus-ada added the `ai` label
- 2026-11-14T17:33:02Z @neo-opus-ada added the `regression` label
- 2026-11-14T17:33:02Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T17:33:28Z @neo-opus-ada cross-referenced by #1710
- 2026-11-14T17:43:03Z @neo-opus-ada cross-referenced by PR #1714
- 2026-11-14T18:01:09Z @tobiu referenced in commit `fdf7f8e` - "fix(memory-core): a tool call no longer re-opens a closed turn (#1713) (#1714)

The Claude PostToolUse hook names the turn its start stored (#1686), so the
progress after add_memory overwrote the closed row with status active and a
fresh 60-minute expiry. The rotation's live-turn guard then held every
Claude sunset for up to an hour.

A named progress or terminal on a terminal row is now the documented no-op.
A named progress past expiresAt still joins its own turn: a tool call that
outlived the TTL is still the turn."
- 2026-11-14T18:01:10Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-14T18:46:37Z

**Post-merge validation: the fix holds live.** The plane was cut to `e44ef997` (#1714 + #1705) with the operator's go. All four services were healthy at 18:44:28Z, and chroma and ingress were untouched.

| | before the cut (my sunset turn, `…3cf11a22`) | after the cut (my turn, `…8da2870a`) |
|---|---|---|
| status | `active` | `terminal` / `completed` |
| last writer | `claude PostToolUse mcp__ccd_turn__wrap_up`, 2.3 s after `add_memory` | `add_memory` at 18:46:09.057Z |
| `expiresAt` | +60 min after the last tool call | the closed row's own; no later write |

The after-cut interval saw two more tool calls after its `add_memory`. Their PostToolUse progress carries the stored `turnId` and is now the documented no-op.

Instrument: a read-only `better-sqlite3` read of my newest `AGENT_TURN_PRESENCE` row on the plane's graph store at 18:46:22Z.

The second AC (a Claude sunset receipt leaves `live-turn` within one lane tick) waits for the next live sunset. #1712 / PR #1720 records that rotation end to end.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-14T18:56:38Z @neo-opus-ada cross-referenced by #1722

