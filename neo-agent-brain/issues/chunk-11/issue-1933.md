---
id: 1933
title: A stalled mc-server names its tool but not the query shape or GC time
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T17:42:33Z'
updatedAt: '2026-11-25T19:07:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1933'
author: neo-opus-ada
commentsCount: 1
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-25T19:07:09Z'
---
# A stalled mc-server names its tool but not the query shape or GC time

## Context

#1843's 2026-11-25 stall lasted 6 minutes, served nothing and held 99% CPU at 1.44 GiB RSS. `get_memory_core_tool_metrics` names the call in flight: `list_messages` `ea337ebb`, started at 16:41:24.927Z, which never finished before the 16:47Z restart ([#1843 comment](https://github.com/neomjs/neo-agent-brain/issues/1843#issuecomment-7169827500)).

It can do that only because `MemoryCoreRecorderService#beginToolCall` persists a start row before dispatch. Two answers are still missing:
- **Which query shape.** The row stores `args_bytes` and, by contract, no argument values.
- **Whether the CPU went to the query or to GC.** Nothing in `ai/` measures event-loop blocks or GC pauses. `git grep` finds no `monitorEventLoopDelay`, `eventLoopUtilization` or `gc` `PerformanceObserver` at `aa7770dd`.

## The Problem

The next stall will again name only a tool. `list_messages` alone takes 3 `box` values × 3 `status` values plus optional filters, and each shape builds a different synchronous query, so the attribution cannot pick a fix.

A 6-minute 99%-CPU hang looks the same whether it is a long synchronous better-sqlite3 scan or V8 mark-compact thrash. The same process died at 14:24Z with `Ineffective mark-compacts near heap limit`.

## The Architectural Reality

- **The start row.** `MemoryCoreRecorderService` (`ai/services/memory-core/MemoryCoreRecorderService.mjs`) inserts it into `mc_tool_call_log` in `beginToolCall`, before dispatch. `buildToolCallRecord` measures `args_bytes` and stores no argument values, and `getMetrics` reports `unfinishedCalls`.
- **Queries block.** better-sqlite3 runs every query synchronously on the event loop. `MailboxService`'s `buildMailboxMatchQuery` unions up to four branches, each with correlated receipt subqueries per row.
- **A lag probe reports late.** A timer-based probe can report a block only after it ends. A block that ends in a restart leaves only the start row, so the start row must carry the shape.
- Design authority: `beginToolCall`'s JSDoc: "A call that wedges the process never reaches the existing completion-only recorder. The unfinished row is therefore the durable evidence: tool identity, start time, and opaque call id, with arguments measured but never stored." This ticket amends that contract, deliberately and only for closed value sets (see the Contract Ledger).

## The Fix

1. **Argument shape on the start row.** A per-tool allow-list of enumerated arguments records their values:
   - `list_messages`: `box`, `status`, `projection`, `includeArchived`, `directOnly`, `limit`;
   - `query_recent_turns`: `detail`, `projection`, `limit`;
   - `who_is_online`: `verbose`.

   Every other argument the caller supplied records its key only. Free text, identities and ids are never stored. The shape goes into a new `args_shape` JSON column, written in `beginToolCall`.
2. **Block and GC attribution after the fact.** Sample event-loop lag and observe `gc` performance entries. When a block longer than a module-constant threshold (no `AiConfig` leaf) ends, write one row with:
   - the block's duration;
   - the GC time inside that window;
   - the ids of the calls in flight across it.
3. **Surface both.** `get_memory_core_tool_metrics` reports `args_shape` on `unfinishedCalls` and slow calls, and lists the recent block rows.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `mc_tool_call_log` start row | `beginToolCall` JSDoc (amended) | adds `args_shape`: allow-listed enumerated values, keys only for everything else | a tool with no allow-list records keys only | `beginToolCall` JSDoc | unit |
| `get_memory_core_tool_metrics` response | `MemoryCoreRecorderService#getMetrics` | `args_shape` on unfinished and slow calls, plus recent block rows | absent fields when telemetry is disabled | tool description in `openapi.yaml` | unit |

## Acceptance Criteria

- [ ] AC-1 (unit): a `list_messages` start row carries `box`, `status` and `limit` values, and only the keys of `fromIdentity`, `taggedConcepts` and `since`. No value outside the allow-list appears anywhere in the raw row.
- [ ] AC-2 (unit): a synchronous block longer than the threshold writes one block row with its duration and the ids of the calls in flight across it. GC time inside the window is counted.
- [ ] AC-3 (unit): `get_memory_core_tool_metrics` returns `args_shape` on `unfinishedCalls` and lists block rows.

## Out of Scope

- Fixing any hot query. Each gets its own sub of #1843 once this probe names it.
- New retention for the block rows: they follow `mc_tool_call_log`'s.

## Avoided Traps

- **Storing the arguments.** The recorder's redaction contract exists because arguments carry message text and identities. Closed value sets and key names name the query shape without that risk.
- **A lag probe alone.** It fires only after a block ends, and today's block ended in a restart. The shape has to ride the start row, which is written before the block.

## Decision Record impact

None found. `learn/agentos/decisions/` has no record on Memory Core tool telemetry (ADR 0013 covers KB ingestion telemetry). The amended contract lives in `beginToolCall`'s JSDoc.

## Related

Parent: #1843 · #1931 (the `who_is_online` floor) · #1929 (headroom, sized after #1843)

Live latest-open + closed sweep:
- At 2026-11-25T17:41Z: the latest 20 open, plus open and closed searches for "event loop blocked stall" and "tool telemetry unfinished call args". Read the hits #73, #54 and #1142: different services and causes. No equivalent found.
- 157 open issues, the red band. #1843 owns the attribution but spans three tools plus this probe, so this is filed as a sub.
- A2A sweep: the last 30 messages, all read states. No claim on this scope.
- MC sweep: "mc-server stall event loop blocked 99% CPU which tool call in flight GC thrash unfinished call", 6 results. Grace's 08-09 diary saw a slow `list_messages` path, but there is no prior decision on a probe.
- Own-assignment sweep: #1843 is the parent, and #1931 sits on another surface.

Retrieval Hint: "mc-server stall attribution args shape event loop block GC tool telemetry"
Origin Session ID: cc55a843-2d87-4222-b61d-8efb38ee837d

## Timeline

- 2026-11-25T17:42:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T17:42:36Z @neo-opus-ada added the `enhancement` label
- 2026-11-25T17:42:36Z @neo-opus-ada added the `ai` label
- 2026-11-25T17:42:37Z @neo-opus-ada added the `performance` label
- 2026-11-25T17:42:37Z @neo-opus-ada added the `agent-os` label
- 2026-11-25T17:42:40Z @neo-opus-ada added parent issue #1843
### @neo-opus-ada - 2026-11-25T17:48:20Z

**Handover (Ada, session rotation; the next Ada session continues as assignee).**

- **Done on branch `ada/1933-stall-attribution`, `a3b7ddae`, pushed, no PR yet:** AC-1 and AC-3.
  - `helpers/toolArgsShape.mjs` reduces a call's arguments to `{values, keys}`. It keeps closed values (booleans, finite numbers, short lower-camel words) for allow-listed arguments and only keys otherwise, so an identity, id or prose never lands, even under an allowed key.
  - `beginToolCall` writes that shape to the new `args_shape` column, and `getMemoryCoreToolMetrics` serves it as `argsShape` on `unfinishedCalls` and `recentSlowCalls`. OpenAPI defines it once as `ToolCallArgsShape`.
  - The recorder and helper specs are green (34/34), and the arms touching the recorder or OpenAPI pass 1656/1657. The one red, `planeConfig.spec.mjs` › `"standalone": "false"` refuses at boot, fails identically on clean `dev`.
- **Open: AC-2, the block and GC probe.** Planned design:
  1. The recorder keeps an in-memory set of in-flight call ids: add in `beginToolCall`, delete on completion in `logToolCall`.
  2. A probe module samples event-loop drift with an unref'd interval. When the lag exceeds a module-constant threshold (about 1 s; no `AiConfig` leaf), it records one row in a new `mc_event_loop_block` table: duration, the GC time from a `PerformanceObserver` (`gc` entries inside the window), and the in-flight ids.
  3. Give the probe a `noteGc(startMs, durationMs)` seam so the GC arm does not depend on forcing a real collection.
  4. Serve recent blocks from `get_memory_core_tool_metrics`, which means adding the key to OpenAPI and to the spec's key pin (`Object.keys(metrics)`), the way `walDrain` was added.
  5. Test with a synchronous busy-wait past the threshold inside a begun call, then await one tick.
- **Then:** open the PR with `Resolves #1933`; the reviewer is a Gemini seat until Thursday's reset, per the operator's dated preference.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-25T17:52:30Z @neo-opus-ada cross-referenced by #1843
- 2026-11-25T18:25:29Z @neo-opus-ada cross-referenced by #1935
- 2026-11-25T18:39:07Z @neo-opus-ada cross-referenced by PR #1936
- 2026-11-25T18:51:27Z @neo-opus-ada referenced in commit `5b400f3` - "feat(memory-core): record event-loop blocks with calls and GC (#1933)

An unref'd probe samples event-loop lag and, one sample after a block of a second or more ends, writes an mc_event_loop_block row: how late the sample ran, the gc performance-entry time inside the window, and the ids of the calls in flight across it, counting a call that returned inside the window, since the blocking call usually completes before the late sample runs. The recorder starts it in initAsync and feeds it from beginToolCall and logToolCall; get_memory_core_tool_metrics serves the rows as eventLoopBlocks."
- 2026-11-25T18:51:36Z @neo-opus-ada cross-referenced by PR #1937
- 2026-11-25T19:07:08Z @tobiu referenced in commit `191aaaa` - "feat(memory-core): name a stall's query shape, calls and GC time (#1933) (#1937)

* feat(memory-core): record each tool call's argument shape (#1933)

The 2026-11-25 16:41Z stall named its in-flight call, list_messages,
only through the start row beginToolCall writes before dispatch. That
row stores no argument values, so it could not say which query shape
held the event loop.

A pure helper, helpers/toolArgsShape.mjs, reduces a call's arguments
to {values, keys}. It keeps the closed values (booleans, finite
numbers, short lower-camel words) of the tool's allow-listed arguments,
and only the keys of everything else the caller supplied. An identity,
an id or prose never lands in the row, even under an allow-listed key.
The start row stores the shape in a new args_shape column, and
get_memory_core_tool_metrics serves it as argsShape on unfinished and
slow calls. OpenAPI defines it once as ToolCallArgsShape.

AC-2, the event-loop block and GC probe, follows on this branch.

* feat(memory-core): record event-loop blocks with calls and GC (#1933)

An unref'd probe samples event-loop lag and, one sample after a block of a second or more ends, writes an mc_event_loop_block row: how late the sample ran, the gc performance-entry time inside the window, and the ids of the calls in flight across it, counting a call that returned inside the window, since the blocking call usually completes before the late sample runs. The recorder starts it in initAsync and feeds it from beginToolCall and logToolCall; get_memory_core_tool_metrics serves the rows as eventLoopBlocks."
- 2026-11-25T19:07:10Z @tobiu closed this issue
- 2026-11-25T19:09:06Z @neo-opus-ada cross-referenced by #1939
- 2026-11-25T19:09:13Z @neo-opus-ada cross-referenced by #1940

