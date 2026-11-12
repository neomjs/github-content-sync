---
id: 1647
title: run-bridge dies by signal on a SIGTERM just after its port listens
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T14:35:38Z'
updatedAt: '2026-11-12T15:32:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1647'
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
closedAt: '2026-11-12T15:32:37Z'
---
# run-bridge dies by signal on a SIGTERM just after its port listens

## Context

Brain PR #1645's `unit` gate went red on one test that its base run passed:

```
ConnectionService.spec.mjs › run-bridge.mjs shuts down cleanly and releases its port on SIGTERM (#1302)
Expected: 0
Received: null
compareUnitFailures: 1 introduced, 0 fixed, 0 pre-existing
```

`null` is the exit code of a process that a signal ended. #1645 changes no file under `ai/mcp/server/neural-link/`.

It is one occurrence, not three. Run 43230881593 attempt 1 ran `suite (head)` once (13:54Z). Attempts 2 and 3 were "re-run failed jobs", which re-ran only the `unit` compare job against attempt 1's reports, so the same verdict came back by construction. Clearing a flake on this workflow takes a full re-run (`gh run rerun <id>` without `--failed`).

## The Problem

`run-bridge.mjs` installs its SIGINT/SIGTERM handlers only after `await Bridge.startServer(...)` resolves. The port accepts connections as soon as `listen()` returns, but `startServer` resolves from the `listening` callback, a tick later. A SIGTERM that lands in that gap meets the default disposition: the process dies by signal instead of closing its server and exiting 0.

The spec polls the port and sends SIGTERM as soon as it answers, so a loaded runner can land in the gap. In production, `ConnectionService#stopBridgeProcess` (`ConnectionService.mjs:1256`) sends the same SIGTERM; the port is freed either way, so there the cost is only the exit status. In CI the cost is a red gate on an unrelated PR.

Reproduced deterministically at `cfb998af`: a preload that sends the process SIGTERM at the moment the server on `NEO_NL_PORT` emits `listening` prints `SIGTERM listeners now: 0`, the process exits 143 (128 + SIGTERM), and the port is released.

## The Architectural Reality

- `ai/mcp/server/neural-link/run-bridge.mjs`: the async IIFE awaits `assertConfigFresh`, `Bridge.ready()` and `Bridge.startServer()`, then defines `shutdown` and registers both handlers.
- `Bridge.startServer` (`Bridge.mjs:119-151`) sets `this.wss` and resolves inside `wss.on('listening')`. `Bridge.stopServer` returns at once while `this.wss` is unset.
- Design authority: neomjs/neo-agent-brain#1302's Contract Ledger, row `run-bridge.mjs`: "On SIGTERM or SIGINT, closes its server and exits 0". The row names no startup exception.
- The same shape (handlers installed only after the listen resolves) exists in `ai/services/fleet/fleetServer.mjs` (`boot()`) and `ai/services/fleet/devFleetServer.mjs` (the `cleanShutdown` registration). Neither has a recorded failure; see Out of Scope.

## The Fix

1. In `run-bridge.mjs`, define `shutdown` and register both handlers before the first `await`. A signal during startup then exits 0 as well: `Bridge.stopServer()` does nothing while `wss` is unset, and `process.exit` releases whatever is half-open.
2. Add a deterministic arm to `ConnectionService.spec.mjs`: spawn `run-bridge.mjs` with a preload that sends SIGTERM when the bridge's server emits `listening` (the reproduction above), then expect exit code 0 and a released port. The existing poll-based arm stays.

## Acceptance Criteria

- [ ] AC-1: `run-bridge.mjs` registers its SIGINT/SIGTERM handlers before its first `await`, with a comment naming the gap they close.
- [ ] AC-2: the deterministic arm fails on `dev` at `cfb998af` (exit by signal) and passes with AC-1.
- [ ] AC-3: the rest of `ConnectionService.spec.mjs`, including the #1302 arm, stays green.

## Out of Scope

- `fleetServer.mjs` and `devFleetServer.mjs`. They have the same shape but no recorded failure, and `devFleetServer`'s shutdown carries the continuity handoff, so moving its registration needs its own witness. Captured as a defect-note.
- The poll-based arm's timing, which is correct once the gap is gone.
- The workflow's "re-run failed jobs" behavior: it is GitHub's, and a full re-run covers it.

## Related

neomjs/neo-agent-brain#1302 (the handlers and the poll-based arm, PR #1305) · PR #1645 (the red).

Live latest-open + closed sweep: latest 20 of 157 open issues (red band; no owning open ticket to fold into) and closed searches (`run-bridge SIGTERM`, `bridge exit code null SIGTERM handler`; #1302 read for its Contract Ledger) at 2026-11-12T14:36Z; no equivalent or exclusion found. A2A in-flight claim sweep: the latest 30 messages, all read states; no claim on this scope. MC sweep: no prior diagnosis of this gap. Own-assignment sweep: 5 open, none on this surface. Structure map: N/A, no new file; the owning folder is `ai/mcp/server/neural-link/`.

Not a `good first issue`: it blocks an open PR's gate class today and is already owned.

Retrieval Hint: "run-bridge SIGTERM before handlers exit code null ConnectionService spec flake compareUnitFailures introduced"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T14:35:38Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T14:35:40Z @neo-opus-ada added the `bug` label
- 2026-11-12T14:35:40Z @neo-opus-ada added the `ai` label
- 2026-11-12T14:35:40Z @neo-opus-ada added the `testing` label
- 2026-11-12T14:39:54Z @neo-opus-ada cross-referenced by PR #1648
- 2026-11-12T14:51:11Z @neo-opus-ada cross-referenced by #1649
- 2026-11-12T15:32:36Z @tobiu referenced in commit `528105e` - "fix(neural-link): run-bridge traps SIGTERM before it listens (#1647) (#1648)

run-bridge.mjs registered its SIGINT/SIGTERM handlers only after await Bridge.startServer(). The port answers as soon as listen() returns, a tick before that promise resolves, so a supervisor's SIGTERM in the gap met the default disposition and ended the process by signal (exit code null) instead of closing the server and exiting 0. CI read it as an introduced failure of the #1302 arm on an unrelated PR.

The handlers now sit above the first await; a signal during startup exits 0 too, since Bridge.stopServer() does nothing while no server exists. A new arm makes the bridge's own server send SIGTERM the moment it emits 'listening': red at cfb998af, green with the move."
- 2026-11-12T15:32:38Z @tobiu closed this issue

