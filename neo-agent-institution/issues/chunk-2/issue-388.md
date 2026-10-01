---
id: 388
title: A closed stdio pipe turns every harness log line into a crash dialog
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T14:14:19Z'
updatedAt: '2026-10-01T14:14:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/388'
author: neo-opus-grace
commentsCount: 0
parentIssue: 7
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
# A closed stdio pipe turns every harness log line into a crash dialog

## Context

2026-10-01, testing #386 on the dev harness. A test driver started `electron main.mjs` with stderr piped to itself, then crashed. The harness kept running, and the operator got a modal dialog over and over:

> A JavaScript error occurred in the main process — Uncaught Exception: Error: write EPIPE … at target.<computed> [as error] (harness/mainLog.mjs:113:17)

The UI-only boot refuses `fleet-request` on every cockpit poll, and main logs each refusal as an error, so the dialogs kept coming until the process was killed.

## The Problem

Reproduced on the harness's own Electron 43.5.0 with a throwaway main. After `app.whenReady`, it writes 20 × `console.error` + `console.log` with stdio piped into a reader that exits at once (`2>&1 | true`). An `uncaughtException` listener records each error instead of letting Electron show it.

| run | uncaught |
|---|---|
| plain `console` | 38 of 40 writes raise `EPIPE` |
| `createMainLog({dir}).install()` | the same 38 |
| control: a live reader (`\| cat >/dev/null`) | 0 |
| a persistent `'error'` listener on `process.stdout` and `process.stderr` | 0 (all 40 dropped by the listener) |

Electron's default handler shows a modal dialog for each uncaught exception, and the harness registers none of its own: `git grep uncaughtException harness/` is empty.

The reader goes away whenever a launcher pipes the harness's stdio and exits first: an agent's shell or test driver, or a pipeline such as `npm start | head`. The installed app is not affected. Its main process, launched from Finder, has stdin, stdout and stderr on `/dev/null` (checked with `lsof` on the running app).

## The Architectural Reality

- `harness/mainLog.mjs` `install(target = console)` wraps `log`, `warn` and `error`. `original(...args)` prints; `write(...args)` tees into `main.log`. The module's contract is that "a log that cannot write stops quietly rather than failing the boot it exists to explain". The file half keeps it (try/catch → `disabled`); the terminal half does not.
- `harness/main.mjs` installs the log at boot in every mode: dev, smoke, packaged.
- `test/playwright/unit/harness/mainLog.spec.mjs` covers the tee with an injected `target`. Nothing covers the terminal streams.

## The Fix

`mainLog.install` attaches one persistent `'error'` listener to each terminal stream: `process.stdout` and `process.stderr`, injectable for the spec the way `target` is.

- A terminal error drops the terminal half for good.
- It records one line in `main.log` the first time, so a later reader of the file knows why the terminal went quiet.
- The file half keeps every line.

## Acceptance Criteria

- [ ] AC-1: a harness main whose stdio reader is gone raises no uncaught exception on console writes. The reproduction above goes from 38 of 40 to 0.
- [ ] AC-2: `main.log` still receives every line, plus one line naming the terminal error, written once.
- [ ] AC-3: unit spec: after `install()`, an error emitted on an injected terminal stream does not throw, and the file records the one line once, however many errors follow.
- [ ] AC-4: with a live terminal nothing changes: lines print and land in the file.

## Out of Scope

- A general `uncaughtException` policy for the main process: what a real uncaught error should do is a separate decision.
- The UI-only boot logging a `fleet-request` refusal on every poll.

## Decision Record impact

`none`.

## Related

#7 (Electron shell), #386 (the testing that hit it).

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T14:13:59Z. No equivalent.
- Exact: `gh search issues --owner neomjs "EPIPE"` found only neo#17094 (closed): the Neural Link stdio witness, not this.
- A2A in-flight sweep: the latest 30 messages, all states. No claim on this surface.
- MC sweep: "harness EPIPE … main process dialog … pipe closed" and "harness mainLog install console tee terminal", 12 results, no prior decision.
- Own-assignment sweep: 3 open (#386, #380, #11), none overlapping.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "harness mainLog EPIPE uncaught dialog closed stdio pipe terminal"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-01T14:14:21Z @neo-opus-grace added the `bug` label
- 2026-10-01T14:14:21Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T14:14:21Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T14:14:22Z @neo-opus-grace added the `ai` label
- 2026-10-01T14:14:28Z @neo-opus-grace added parent issue #7
- 2026-10-01T14:18:15Z @neo-fable-clio cross-referenced by #389
- 2026-10-01T14:18:32Z @neo-opus-grace cross-referenced by PR #390
- 2026-10-01T14:18:43Z @neo-fable-clio cross-referenced by #391
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392
- 2026-10-01T16:28:58Z @neo-opus-grace cross-referenced by #396

