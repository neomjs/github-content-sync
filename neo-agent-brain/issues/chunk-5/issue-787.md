---
id: 787
title: who_is_online re-reads every wake receiver record on each call
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-03T06:54:03Z'
updatedAt: '2026-10-03T09:49:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/787'
author: neo-opus-grace
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
closedAt: '2026-10-03T08:41:15Z'
---
# who_is_online re-reads every wake receiver record on each call

## Context

Found on 2026-10-03 while timing the cold `get_graph_scene` read (neomjs/neo-agent-institution#312, gap 2). The plane's 50 most recent slow calls (> 2 s, 06:12–06:41Z, `get_memory_core_tool_metrics`) were 36 `who_is_online` (avg 26.4 s, max 163.6 s), 11 `healthcheck` (avg 22.4 s, max 82.0 s), 2 `query_recent_turns` and 1 `list_messages`, while the team's seats booted.

Measured inside `neo-local-agent-os-mc-server-1` (Brain `804356b`), with a separate node process doing what `readWakeDelivery()` does against `/app/wake-receiver-records`:

| Read | Time |
|---|---|
| `readdir`, 9,562 entries | 0.19–0.21 s |
| sequential `readFile` + `JSON.parse`, 9,562 files, 17.0 MB | 3.76–3.86 s |
| one full read, solo | 3.9–4.4 s |
| three full reads at once | 7.6 s each (wall 7.65 s) |

The directory is a macOS bind mount (`~/Library/Application Support/Neo/AgentOS/wake/state/records` → `/app/wake-receiver-records`). It holds 9,562 files (37 MB on disk); the oldest is from 2026-08-16, about 330 were written in the last day, and nothing prunes it.

Observed: the per-read cost and how it scales under concurrency. Inferred, not measured: that this read is most of the 26 s `who_is_online` average. That call also scans subscriptions and the A2A trail.

## The Problem

Every `who_is_online` call pays a full read of the receiver's history:
- `WakeSubscriptionService._readWakeReachability` calls `readWakeDelivery()` on each call (`WakeSubscriptionService.mjs:933`), with no cache and no shared in-flight read.
- `readWakeDelivery()` (`wakeDeliveryReader.mjs:49`) reads every `.json` record, one after another.
- Callers include every seat's session start and claim sweep, plus each fleet server's presence read (`planeWhoIsOnlineReader.mjs:23`, wired at `devFleetServer.mjs:273` and `:301`).
- Overlapping calls each repeat the read over the same bind mount and the same 4-thread libuv pool. A burst slows every caller, and every other file read in the Memory Core process.
- The cost grows with the directory, and the directory only grows.

`healthcheck` reaches the same reader only on a full check (`HealthService.mjs:534`). That check is single-flighted and cached for five minutes while healthy, but runs on every call while degraded.

## The Architectural Reality

- `wakeDeliveryReader.mjs` is the I/O half and `wakeDeliveryProjection.mjs` the pure half (`projectWakeDelivery(records)`). The split stays.
- **A terminal record never changes.** `receiverState.mjs:17` defines `TERMINAL_STATES` as `delivered`, `skipped`, `failed` and `unknown`. `transition()` (`:167`) rewrites a record only when its current state equals the caller's expected state, and every caller expects `pending` or `dispatching` (`receiver.mjs:245, 266, 346, 359, 377`; `receiverState.mjs:196`).
- **A reader never sees a partial file.** A record is created by `link` and replaced by an atomic `rename` (`:232`).
- The reader's contract has four outcomes (`unconfigured`, `no-records`, `unreadable`, `observed`) and one rule: a malformed record is skipped.

## The Fix

In `wakeDeliveryReader.mjs`, per declared directory:
1. **One read in flight.** Overlapping calls join the read already running.
2. **Read only what can have changed.** Keep each parsed record whose state is terminal, keyed by file name. On the next call, `readdir`, then read only the names not held, plus held names whose state is non-terminal. Drop held names the directory no longer lists.
3. **The contract is unchanged.** The four outcomes and skip-malformed stay. The projection still receives full records, so its field set remains the projection's own business.
4. **The writer makes finality a rule.** *(Added 2026-10-03 at implementation.)* `receiverState.mjs` exports `TERMINAL_STATES`, and `transition()` refuses an expected state that is terminal. A terminal record is therefore final by construction, not only because every caller happens to use it that way.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `readWakeDelivery()` | `wakeDeliveryReader.mjs:49` | The same verdicts. A repeat call reads only new or non-terminal records, and overlapping calls share one read. | A directory that turns unreadable answers `unreadable`; held records are never served as `observed`. | JSDoc | unit |
| `who_is_online` wake axis | `WakeSubscriptionService._readWakeReachability` | unchanged output | unchanged | — | existing arms |
| `TERMINAL_STATES` and `transition()` *(added 2026-10-03)* | `ai/daemons/wake/receiverState.mjs` | `TERMINAL_STATES` (`delivered`, `skipped`, `failed`, `unknown`) is exported. `transition()` throws when the expected state is terminal, so no record leaves a terminal state. The non-terminal transitions are unchanged: `pending → dispatching` (`receiver.mjs:245`), the context-gate deferral `dispatching → pending` (`:266`, `:359`), the outcome `dispatching → terminal` (`:346`, `:377`), and `recoverInterrupted()`'s `dispatching → unknown`. **Consumer:** `readWakeDelivery()` holds terminal records by file name, so it depends on this finality. | A caller that tries to move a terminal record gets a loud error and the file is unchanged; there is no silent rewrite the reader would miss. | JSDoc on `TERMINAL_STATES` | unit (`receiverState.spec.mjs`, the finality arm; mutation-killed) |

## Acceptance Criteria

- [ ] AC-1: A terminal record is read once. If its file is rewritten between two calls, the second verdict still reflects the held record (unit).
- [ ] AC-2: A non-terminal record is re-read: a `pending` record that turns `delivered` between calls shows in the second verdict (unit).
- [ ] AC-3: A removed file leaves the next verdict (unit).
- [ ] AC-4: Two overlapping calls perform one directory read (unit).
- [ ] AC-5: The four outcomes and skip-malformed hold, including a directory that turns unreadable after a held read (unit).
- ~~AC-6 *(post-merge, after the next container refresh)*: on the plane, a repeat read costs about one `readdir`, measured with the same in-container probe.~~ Moved to PMV-1, since a PR cannot deliver it.

## Post-Merge Validation

- [x] PMV-1: a repeat read costs about one `readdir` on the plane's records. Discharged before merge (2026-10-03): the shipped module was run in a throwaway container from the plane's MC image, over the real bind-mounted records, read-only. The first read took 2,711 ms, repeats 195–197 ms, and three overlapping calls 204 ms sharing one read. Receipt in the PR's Test Evidence.

## Out of Scope

- **Pruning the receiver's records** (`receiverState.mjs`). After this fix a repeat call costs one `readdir` (0.2 s at 9,562 entries) plus the held records' memory. Retention earns its own leaf when that measures slow. Not filed.
- **The cold `get_graph_scene` read.** Measured separately (3.5–4.0 s in-container); the findings go to neomjs/neo-agent-institution#312.
- `healthcheck`'s own cadence and cache.

## Avoided Traps

- **A TTL cache.** It serves a stale verdict for its whole window, and the wake axis exists to catch failing dispatches.
- **Holding records by file name without the terminal rule.** `pending` and `dispatching` records are replaced in place.
- **Holding only the projection's current fields.** That couples the I/O half to the projection's field set.

## Decision Record impact

None. Structure map: N/A. This is an in-place change to `ai/services/memory-core/wakeDeliveryReader.mjs` (owning folder `ai/services/memory-core`); no new file.

## Related

#503 (the delivery verdict's correctness) · #512, #547 and #734 (the reader's history) · neomjs/neo-agent-institution#312 (where this was found).

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-03T06:53:12Z; no equivalent found. A2A sweep: the last 30 messages, all read states; no claim on this scope. MC sweep: "who_is_online slow wake receiver records readdir every call healthcheck latency bind mount", 6 results, no prior decision found. Own-assignment sweep: 21 open; none on this surface (#68 and #69 are daemon-side).

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "who_is_online slow wake receiver records readWakeDelivery bind mount 9562 files"

🖖 Grace (Claude Opus 5.5, Claude Code)



## Timeline

- 2026-10-03T06:54:03Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-03T06:54:04Z @neo-opus-grace added the `bug` label
- 2026-10-03T06:54:04Z @neo-opus-grace added the `ai` label
- 2026-10-03T06:54:04Z @neo-opus-grace added the `performance` label
- 2026-10-03T06:54:05Z @neo-opus-grace added the `agent-os` label
- 2026-10-03T06:59:37Z @neo-fable cross-referenced by #788
- 2026-10-03T07:08:01Z @neo-opus-grace cross-referenced by PR #791
- 2026-10-03T07:08:47Z @neo-opus-grace cross-referenced by #312
- 2026-10-03T08:41:15Z @tobiu referenced in commit `1dd52f3` - "fix(memory): the wake delivery reader holds terminal records and shares one read (#787) (#791)

who_is_online read all 9,562 wake receiver records on every call:
4.4 s solo in the MC container, 7.6 s each at three overlapping.
A terminal record's file is final, so the reader now holds it by
file name and reads only new, pending or dispatching records;
overlapping calls join one read. receiverState exports
TERMINAL_STATES and refuses a transition from a terminal state,
which is what makes the hold safe."
- 2026-10-03T08:41:15Z @tobiu closed this issue
- 2026-10-03T08:48:55Z @neo-opus-vega cross-referenced by #486
### @neo-opus-grace - 2026-10-03T09:49:28Z

**Deployed receipt.** The canonical plane runs Brain `fb40366` since 09:41:58Z, and that cut includes #791. One `who_is_online` there at 09:49:00Z took **392 ms** server-side.

The same verb took 9.2–20.1 s (5 calls, avg 16.1 s) between 09:33Z and 09:37Z on the previous image, `804356b`. The first scene read after the restart took 2.6 s, no longer queued behind it (Institution #486).

🖖 Grace (Claude Opus 5.5, Claude Code)


