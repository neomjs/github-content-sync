---
id: 1255
title: Wake receiver default onStuck deadlocks in uv__threadpool_cleanup
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-24T21:07:01Z'
updatedAt: '2026-10-24T21:32:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1255'
author: neo-gemini-maryam
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
closedAt: '2026-10-24T21:32:16Z'
---
# Wake receiver default onStuck deadlocks in uv__threadpool_cleanup

## Context

On 2026-10-24 at `09:03:09.662Z`, the host wake receiver (`com.neomjs.agent-os-wake`, Node `25.9.0_1` / `libuv 1.52.1`) tripped its 15-second `reload` watchdog (`[Wake Receiver] STUCK: reload unsettled for 18379 ms (bound 15000 ms)`), synchronously persisted `{"step":"reload","at":"2026-10-24T09:03:09.662Z"}` to `records/receiver.liveness`, and invoked the default `onStuck` callback (`() => process.exit(1)` in `ai/daemons/wake/receiver.mjs:543`). Instead of terminating so `launchd` (`KeepAlive: true`, `ThrottleInterval: 10`) could respawn a fresh process within ~10 seconds as designed in `#836` (`de222e4f`), the process wedged inside `process.exit(1)` for 11.8 hours (`09:03Z`–`20:52Z`) with `TCP 127.0.0.1:3199 (LISTEN)` still bound in the kernel and its V8/libuv event loop completely stopped. Every `mc-server` webhook delivery batch timed out and degraded active seat routes until the wedged PID was terminated with `SIGKILL`.

- `Live latest-open sweep:` checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1254`..`#936`) at `2026-10-24T21:05:24Z`; no equivalent found.
- `A2A in-flight claim sweep:` checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-24T21:05:27Z`; no competing `[lane-claim]` or `[lane-intent]` found.
- `MC sweep:` `query_raw_memories({ query: "neo-agent-brain wake receiver reload unsettled stuck onStuck process.exit timeout route degraded", nResults: 6 })` + `ask_knowledge_base({ query: "wake receiver onStuck process.exit reload unsettled stuck watchdog", type: "ticket" })`, 6 + 5 results; `#836` (`de222e4f`) and `#841` (`7f895e81`) are the closed predecessors that introduced the watchdog and liveness writer, with no prior ticket addressing the `uv__threadpool_cleanup` exit deadlock.
- `Own-assignment sweep:` 0 open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-brain`.
- `Structure Map Gate:` ran `npm run ai:structure-map -- --files --loc`; owning directory is `ai/daemons/wake/` (`receiver.mjs`, 513 codeLoc; `receiverLiveness.mjs`, 96 codeLoc).

## The Problem

`#836` (`PR #838`, `de222e4f`) placed `accept`, `reload`, `drain`, and `dispatch` under a step watchdog so that an unsettled step logs `[Wake Receiver] STUCK: ...` and calls `onStuck` to exit non-zero for supervisor restart. `#841` (`PR #851`, `7f895e81`) added `liveness.stuck(stuck)` immediately before `onStuck(stuck)` so `records/receiver.liveness` records the stuck step synchronously via `writeFileAtomicSync` before termination.

In the 2026-10-24 incident, sampling the wedged receiver process (`/usr/bin/sample <pid> 1`) while it held `127.0.0.1:3199` revealed the exact main-thread call stack:

```text
main -> node::Start -> ... -> node::Environment::Exit(node::ExitCode)
  -> node::DefaultProcessExitHandlerInternal(node::Environment*, node::ExitCode)
    -> uv_library_shutdown -> uv__threadpool_cleanup
      -> uv_thread_join -> _pthread_join -> __ulock_wait
```

In Node `25.x` (`libuv 1.52+`), `process.exit(1)` invokes `uv_library_shutdown()`, which calls `uv__threadpool_cleanup()` and `uv_thread_join()` on every worker thread in the `libuv` threadpool. Because `reload`, `accept`, and `drain` are `node:fs/promises` operations backed by the `libuv` threadpool, when `onStuck` fires *because* one of those `libuv` filesystem tasks never settled, at least one `libuv` worker thread is blocked in an unsettled kernel I/O call. Calling `process.exit(1)` therefore blocks forever in `uv_thread_join()`, preventing process termination, keeping the listening socket open in the kernel with a dead event loop, and defeating `launchd` `KeepAlive` recovery.

## The Architectural Reality

- `Design authority:` `"A stuck step leaves nothing to catch, and its owner is the supervisor: the production onStuck exits so the LaunchAgent restarts a fresh process."` (`ai/daemons/wake/receiver.mjs:261-262`, `ai/daemons/wake/receiver.mjs:531`, and `ai/daemons/wake/receiverLiveness.mjs:206`: `"Records a stuck step. Synchronous, because the process exits right after it."`)
- In `ai/daemons/wake/receiver.mjs:535-582` (`startWakeReceiver`):
  - Line `269`: `createWakeReceiver`'s watchdog logs `[Wake Receiver] STUCK: ...` via `logger.error`.
  - Line `543`: `onStuck` defaults to `() => process.exit(1)`.
  - Lines `578-581`: `createWakeReceiver` wraps `onStuck` as:
    ```javascript
    onStuck: stuck => {
        liveness.stuck(stuck);
        onStuck(stuck)
    }
    ```
- `liveness.stuck(stuck)` (`ai/daemons/wake/receiverLiveness.mjs:211-218`) writes `records/receiver.liveness` synchronously on the main thread via `writeFileAtomicSync` (`ai/services/shared/atomicFileWrite.mjs:93-115`, which uses synchronous `fs.openSync` / `fs.writeFileSync` / `fs.fsyncSync` / `fs.renameSync` on the main thread and does not touch the `libuv` threadpool).
- Neither `startWakeReceiver` nor `WakeReceiverState` registers any `process.on('exit')` cleanup hook; crash-interrupted `dispatching` records are recovered on the next startup via `state.recoverInterrupted()` (`ai/daemons/wake/receiver.mjs:561`, `ai/daemons/wake/receiverState.mjs:202-214`).

## The Fix

1. In `ai/daemons/wake/receiver.mjs`:
   - Replace the default `onStuck = () => process.exit(1)` in `startWakeReceiver` with a hard self-signal termination (`() => process.kill(process.pid, 'SIGKILL')`) after `liveness.stuck(stuck)` has completed its synchronous main-thread atomic write.
   - Forward `stepBoundsMs = STEP_BOUNDS_MS` and `watchdogIntervalMs = STEP_WATCHDOG_INTERVAL_MS` from `startWakeReceiver` to `createWakeReceiver` as test seams so `startWakeReceiver`'s watchdog-to-`liveness.stuck`-to-`SIGKILL` sequence can be exercised directly in unit tests.
   - Update the JSDoc on `startWakeReceiver` (`options.onStuck`, `options.stepBoundsMs`, `options.watchdogIntervalMs`).
2. In `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs`:
   - Add unit test coverage verifying that when a watched step in `startWakeReceiver` exceeds its bound, `receiver.liveness` is synchronously stamped with the `stuckExits` entry before `onStuck` fires, and that the default `onStuck` terminates via `process.kill(process.pid, 'SIGKILL')` rather than `process.exit(1)`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `startWakeReceiver` default `onStuck` (`ai/daemons/wake/receiver.mjs:545, 582-585`) | `#836` (`de222e4f`) & `#841` (`7f895e81`); `ai/daemons/wake/receiver.mjs:261-262, 531` | After `liveness.stuck(stuck)` synchronously persists `stuckExits` via `writeFileAtomicSync`, default `onStuck` calls `process.kill(process.pid, 'SIGKILL')` so the process terminates immediately without entering `uv__threadpool_cleanup` | Custom `options.onStuck` injection seam remains unchanged for callers/tests | JSDoc in `ai/daemons/wake/receiver.mjs` | L1 unit spec in `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs` |
| `startWakeReceiver` `stepBoundsMs` & `watchdogIntervalMs` options (`ai/daemons/wake/receiver.mjs:547-548, 578-579`) | `createWakeReceiver` in `ai/daemons/wake/receiver.mjs:214-216, 228-229` | Forwarded to `createWakeReceiver` with production defaults `STEP_BOUNDS_MS` and `STEP_WATCHDOG_INTERVAL_MS` so tests can exercise `startWakeReceiver` watchdog termination at millisecond bounds | Defaults to `STEP_BOUNDS_MS` and `STEP_WATCHDOG_INTERVAL_MS` | JSDoc in `ai/daemons/wake/receiver.mjs` | L1 unit spec in `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs` |

## Decision Record impact

`none` — bug fix restoring the supervisor-restart invariant established by `#836` and `#841`.

## Acceptance Criteria

- [ ] `startWakeReceiver` in `ai/daemons/wake/receiver.mjs` terminates via `process.kill(process.pid, 'SIGKILL')` in its default `onStuck` handler after `liveness.stuck(stuck)` completes synchronously, avoiding `uv__threadpool_cleanup` (`uv_thread_join`) deadlocks on hung `libuv` threadpool steps.
- [ ] `startWakeReceiver` forwards `stepBoundsMs` and `watchdogIntervalMs` test seams to `createWakeReceiver` and updates JSDoc in `ai/daemons/wake/receiver.mjs`.
- [ ] Unit test in `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs` verifies that `liveness.stuck` persists `stuckExits` on disk before `onStuck` runs and that the default `onStuck` invokes `process.kill(process.pid, 'SIGKILL')`.

## Out of Scope

- Changing `STEP_BOUNDS_MS` thresholds (`accept: 15s`, `reload: 15s`, `drain: 60s`, `dispatch: 30s`).
- Modifying `createReceiverLiveness` in `ai/daemons/wake/receiverLiveness.mjs` (its `stuck()` method already uses synchronous main-thread `writeFileAtomicSync`).

## Avoided Traps / Gold Standards Rejected

- **Standard `process.exit(1)` inside a `libuv` I/O watchdog (`Gold Standard Rejected`):** In ordinary Node CLI tools, `process.exit(1)` is the standard non-zero exit primitive. In a watchdog guarding `node:fs/promises` steps on Node 25 (`libuv 1.52+`), `process.exit(1)` runs `uv_library_shutdown() -> uv__threadpool_cleanup() -> uv_thread_join()`, which deterministically deadlocks whenever the watchdog fired because a `libuv` worker thread hung in kernel I/O. Because `liveness.stuck()` already flushes `receiver.liveness` synchronously on the main thread before `onStuck` is called, `process.kill(process.pid, 'SIGKILL')` is safe and guarantees immediate kernel process teardown and socket close.

## Related

- Follow-up to `#836` (`PR #838`, `de222e4f`) and `#841` (`PR #851`, `7f895e81`).
- Related incident history: `#503`.

Origin Session ID: e84d121d-df41-4a11-8e45-536824ab4347
Retrieval Hint: "Wake Receiver STUCK reload unsettled uv__threadpool_cleanup process.exit deadlock"
Retrieval Hint: Commit SHA de222e4f..7f895e81


## Timeline

- 2026-10-24T21:07:01Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-24T21:07:02Z @neo-gemini-maryam added the `bug` label
- 2026-10-24T21:07:02Z @neo-gemini-maryam added the `ai` label
- 2026-10-24T21:07:02Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-24T21:13:16Z @neo-gemini-maryam cross-referenced by PR #1256
- 2026-10-24T21:32:16Z @tobiu referenced in commit `10d9104` - "fix(wake): kill wedged receiver with SIGKILL on stuck step (#1255) (#1256)"
- 2026-10-24T21:32:16Z @tobiu closed this issue

