---
id: 757
title: The Claude turn-presence hook blocks every prompt and tool call
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T14:25:51Z'
updatedAt: '2026-10-02T16:15:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/757'
author: neo-opus-ada
commentsCount: 0
parentIssue: 67
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-02T16:15:32Z'
---
# The Claude turn-presence hook blocks every prompt and tool call

## Context

The Claude event manifest every Claude seat is projected from (`ai/scripts/lifecycle/hooks/claude/events.manifest.json`) registers `turnPresenceHook` synchronously on `UserPromptSubmit` and `PostToolUse`. The harness therefore waits for it before each prompt reaches the model and after each tool call. The hook resolves the plane leaves, then either records turn presence over MCP or names a skip. It returns nothing the harness acts on.

Measured 2026-10-02 on `@neo-opus-ada`'s seat. Claude Code's session transcripts record each synchronous hook run with its `durationMs`:

- 688 sessions, 2026-07-16 to 2026-10-02: 305,529 runs and **10.92 h** of blocked wall clock (p50 125 ms, p95 210 ms, max 1,887 ms).
- A recount minutes later covered 305,541 runs and found **none** that recorded presence:
  - 199,945 "no Memory Core plane is configured"
  - 62,553 non-blocking errors (module not found)
  - 22,136 "no NEO_AGENT_IDENTITY in the hook environment"
  - about 20,900 "Cannot find package/module"
  - a few dozen others
- The current session alone: 1,103 `PostToolUse` runs, 161.9 s.

## The Problem

Presence is an enhancement, never a precondition. The hook's own contract says it never fails the session. Waiting on it is pure latency: about 125 ms before every model step that follows a tool call.

The wait grows exactly when the hook starts to work. Once a seat's hook processes resolve a plane (neomjs/neo-agent-brain#752's open post-merge question), each run becomes a full MCP exchange (connect, `initialize`, `tools/call`) on every tool call. Only the writer's 1,500 ms deadline bounds it, under a 2 s ceiling.

#67 named the choice for the remote-plane case: raise a ceiling that fires on every tool call, or stop doing a synchronous network write from a per-tool-use hook. The harness documents the second option: `"async": true` runs a command hook "in the background without blocking", and once it runs "Claude Code doesn't enforce `timeout` on it" (hooks reference, *Run hooks in the background*).

## The Architectural Reality

- **Delivery.** `projectSeatHooks.reconcileClaudeEvents` retires every projector-owned entry in a seat's `.claude/settings.json` and re-appends the manifest's. A changed flag on an existing entry therefore reaches each seat on its next projection; until then `projectSeatHooks --check` reports the drift as entries "to (re)apply".
- **The bound.** `recordTurnPresenceOverMcp` enforces one deadline across all stages: each stage gets what is left. `TurnPresenceHookWriter` feeds it `TurnPresenceConfig.hookWriteTimeoutMs`, which is 1500. With no harness timeout on an async hook, that deadline is the only bound on the background process, and it already is a hard bound.
- **Concurrency.** The harness starts one process per firing and does not deduplicate async runs. Each run ends within its deadline, so overlap is bounded by tool-call rate × 1.5 s.
- **Ordering.** `TurnPresenceService.recordTurnPresence` mints a new interval on `start` and joins the newest active one on `progress`. A `progress` with no open interval is a no-op, so a late progress beacon after completion changes nothing. **A late `start` is not benign** (@neo-gpt-emmy's review of PR #758, reproduced on the service). Completion (`add_memory`) finds no interval to close, and the start then opens a fresh one for a finished turn, which reads active for the whole freshness horizon. The start therefore stays synchronous, so it lands before the turn can complete, and only progress moves to the background.
- **Visibility.** Stderr from a hook that exits 0 "goes to the debug log only, never the transcript" (hooks reference, *Exit code 0*), though the session JSONL carries it today as `hook_success` attachment metadata. Async completion notifications are "suppressed by default" and shown in verbose mode. Where an async run's stderr lands in the JSONL is undocumented and unobserved: no async hook has run on this machine. The post-merge check records it.
- **Untouched.** Every other Claude registration stays synchronous: `wakeArmingHook` and `seatProjectionCheck` (`SessionStart`) and `laneStateStopHook` (`Stop`).
- **Structure map.** The owning folder is `ai/scripts/lifecycle/hooks/claude`. `events.manifest.json` is edited in place, and no file is added.

## The Fix

In `events.manifest.json`:
- The `PostToolUse … progress` entry gains `"async": true` and drops `"timeout": 2`. The harness does not enforce a timeout on an async hook, so it would read as the bound when it is not.
- The `UserPromptSubmit … start` entry stays synchronous at `"timeout": 2`, once per prompt. It opens the turn's interval and must land before the turn can complete.
- The `$comment` states both reasons and names the writer's deadline as the only bound on a background run.
- The commands stay byte-identical.

Specs:
- `turnPresenceHook.spec.mjs` pins progress as async with no timeout and start as synchronous at 2 s, and shows reconciliation replacing a seat's synchronous progress entry while start is unchanged.
- `TurnPresenceService.spec.mjs` pins the terminal-before-start hazard.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `turnPresenceHook` registrations in `events.manifest.json` (existing) | hooks reference, *Run hooks in the background*; this ticket | `PostToolUse` progress: `"async": true`, no `timeout`. `UserPromptSubmit` start: unchanged, synchronous at 2 s. Commands unchanged | A seat not yet re-projected keeps the synchronous progress entry, and `projectSeatHooks --check` names the drift | manifest `$comment` | AC-1, AC-2 |
| The background run's bound: `TurnPresenceConfig.hookWriteTimeoutMs` (existing, 1500 ms) | `recordTurnPresenceOverMcp`'s one-deadline contract | Unchanged; it becomes the only bound | Against an unreachable plane, each run ends at its deadline with a named skip | manifest `$comment` | AC-1 |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] AC-1: in `events.manifest.json`, the `PostToolUse` `turnPresenceHook` entry declares `"async": true` and no `timeout`, while the `UserPromptSubmit` entry stays synchronous at `"timeout": 2`. Commands are byte-identical. The `$comment` names the writer's deadline as the only bound on a background run, and says why start stays synchronous.
- [ ] AC-2: a spec fails if progress loses `async` or regains a `timeout`, or if start becomes async. A reconciliation spec shows a seat's synchronous progress entry replaced by the async one with start unchanged. A service spec pins the terminal-before-start hazard.

## Post-Merge Validation

- [ ] On a re-projected Claude seat, a new session's transcript carries no synchronous `turnPresenceHook` `durationMs` on `PostToolUse`. The check records what an async run leaves in the session JSONL.

Residual-Owner: #67

## Out of Scope

- The budget value for a remote plane, which is #67's remaining question. Async removes the 2 s ceiling for the progress beacon; the start keeps it. Sizing the budget, and bounding background runs against an unreachable plane, stay with #67.
- Plane environment for hook processes on operator seats: neomjs/neo-agent-brain#752's post-merge question.
- The Kimi and Codex presence hooks.

## Avoided Traps

- **Raising `timeout`:** keeps the wait on every tool call.
- **A throttle inside the hook:** the harness still waits on one process per tool call. A bare `node` spawn measured 40–50 ms on this machine before any check could run.
- **Dropping the `PostToolUse` beacon:** presence would then freshen only once per prompt.
- **An async start:** it can land after its turn completed and open an interval for a finished turn. This was the first head of PR #758.

## Related

#67 (parent) · neomjs/neo-agent-brain#752 · #562

Sweeps at 2026-10-02T14:25:31Z:
- **Live latest-open:** the latest 20 open Brain issues, no equivalent.
- **Keyword:** `turn presence hook` and `async hook` across the org; #67 is the only related ticket.
- **A2A:** the last 30 messages, all read states (12:32Z–14:23Z), no claim on this surface.
- **Memory Core:** "turn presence hook blocks every tool call latency PostToolUse synchronous async background", 6 results, no prior decision.
- **Own assignments:** #67 is the parent; its amended premise names this option. #562's PR #752 touches the same manifest only in `$comment`, `SessionStart` and `Stop`.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83
Retrieval Hint: "turn presence hook async blocks every tool call PostToolUse latency"

⚖️ Ada (Claude Opus 5.5, Claude Code)



## Timeline

- 2026-10-02T14:25:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T14:25:53Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T14:25:53Z @neo-opus-ada added the `ai` label
- 2026-10-02T14:25:53Z @neo-opus-ada added the `performance` label
- 2026-10-02T14:25:53Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T14:25:58Z @neo-opus-ada added parent issue #67
- 2026-10-02T14:33:17Z @neo-opus-ada cross-referenced by PR #758
- 2026-10-02T14:47:36Z @neo-opus-ada cross-referenced by #67
- 2026-10-02T15:10:31Z @neo-opus-ada referenced in commit `c7e7bf7` - "fix(hooks): turn presence keeps its start ahead of the turn's completion; only progress runs in the background (#757)

An async start can land after its turn has completed. Completion (add_memory)
then finds no interval to close, and the late start opens a fresh active one
for a finished turn, which reads active for the whole freshness horizon. Emmy
reproduced this on TurnPresenceService.

The start registration goes back to synchronous within its 2 s bound, once
per prompt. Only the per-tool-call progress beacon stays async; a late
progress after completion is a documented no-op.

The hook spec pins progress as async with no timeout, and start as
synchronous at 2 s. The service spec pins the terminal-before-start hazard
that the synchronous start guards against."
- 2026-10-02T16:15:32Z @tobiu referenced in commit `5d13928` - "fix(hooks): the Claude turn-presence hook runs in the background (#757) (#758)

* fix(hooks): the Claude turn-presence hook runs in the background (#757)

Both turnPresenceHook registrations in the Claude event manifest now run
"async": true and carry no timeout. Presence is never a precondition, so no
prompt or tool call waits on it. The harness enforces no timeout on an async
hook, which leaves the writer's one MCP deadline (hookWriteTimeoutMs, 1.5 s)
as the only bound on each run.

Measured on @neo-opus-ada's seat: 305,529 synchronous runs across 688
sessions blocked 10.92 h (p50 125 ms), and none of them recorded presence.

projectSeatHooks retires and re-appends owned entries, so each seat picks up
the flag on its next projection. The spec pins both registrations and the
reconciliation that replaces a seat's synchronous entries.

* fix(hooks): turn presence keeps its start ahead of the turn's completion; only progress runs in the background (#757)

An async start can land after its turn has completed. Completion (add_memory)
then finds no interval to close, and the late start opens a fresh active one
for a finished turn, which reads active for the whole freshness horizon. Emmy
reproduced this on TurnPresenceService.

The start registration goes back to synchronous within its 2 s bound, once
per prompt. Only the per-tool-call progress beacon stays async; a late
progress after completion is a documented no-op.

The hook spec pins progress as async with no timeout, and start as
synchronous at 2 s. The service spec pins the terminal-before-start hazard
that the synchronous start guards against."
- 2026-10-02T16:15:32Z @tobiu closed this issue

