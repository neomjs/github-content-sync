---
id: 219
title: The packaged shell keeps no log of its own boot
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T20:49:30Z'
updatedAt: '2026-09-25T22:34:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/219'
author: neo-opus-ada
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
closedAt: '2026-09-25T22:34:09Z'
---
# The packaged shell keeps no log of its own boot

## Context

On 2026-09-25 the operator's packaged shell was launched from Finder at 19:53Z with no plane record (Neural Link session `ac07d3c8`, `neo-harness/0.0.1`). Read through the Neural Link at 20:45Z, 52 minutes later, the cockpit still reported:
- `daemonState: degraded`, `daemonDegradedReason: boot-not-ready`, `shellTransport: null`;
- the static sample roster, with every fleet call refused ("fleet: Brain is not ready").

`appLifecycle.settleBrainBoot(false)` produced that state, and `fleetCapability` then refuses on `boot.up !== true`. **Why the boot failed cannot be read anywhere.** The lines that say it go to main's stdout: `HARNESS_BRAIN_PLAN` (mode, planeBase, startFleet, startOrchestrator), `HARNESS_BRAIN_MODE`, and the Brain children's `HARNESS_BRAIN` lines. A Finder launch has no terminal, so those lines are gone.

## The Problem

The harness persists no main-process log. `harness/` contains no file writer (no `createWriteStream` or `appendFile`, no `app.getPath('logs')`), and `~/Library/Logs` holds nothing for the app. Every field failure of the product boot is therefore a blind one, whether own-mode, attach or plane-attach. The operator sees "Brain is not ready", and no seat can find the cause without reproducing it in a terminal. For plane-attach, that would mean handling the operator's PAT, which no seat does.

## The Architectural Reality

- `harness/main.mjs` prints its diagnostics with `console.log`, all with the `HARNESS_` prefix. `brainLog(line)` already redacts any Brain child line that carries `fleetBearerToken`.
- The packaged shell holds a second secret since Institution #212: the plane bearer (`NEO_FLEET_PLANE_BEARER`, from the `safeStorage` record or the env). Nothing today redacts it from a line.
- Electron resolves `app.getPath('logs')` to `~/Library/Logs/neo-harness` on macOS, the platform's place for an app's logs.
- Precedent for a pure module unit-tested beside the harness: `harness/planeConfig.mjs` with `planeConfig.spec.mjs`, packaged through `electron-builder.yml`'s `files` list and pinned by `pack.spec.mjs`.

## The Fix

1. **`harness/mainLog.mjs`.** `createMainLog({dir, fsModule, maxBytes, secrets})` appends timestamped lines to `<dir>/main.log`. It rotates once to `main.log.1` past `maxBytes` (1 MB), and never throws into the caller.
   - `secrets()` returns the bearers currently held: the fleet bearer, plus the plane bearer from the record or the env.
   - A line containing any of them is written as `[secret-bearing line redacted]`, the same rule `brainLog` applies on stdout.
2. **`main.mjs`.** At startup, main tees its own `console.log` / `console.warn` / `console.error` through the log, so every `HARNESS_*` line of a boot reaches disk, including relayed Brain child lines. This applies in packaged and checkout runs alike.
3. **Packaging.** Add `mainLog.mjs` to `electron-builder.yml`'s `files` and to `pack.spec.mjs`'s module list.

## Acceptance Criteria

- [ ] A packaged launch leaves `~/Library/Logs/neo-harness/main.log` carrying that boot's `HARNESS_BRAIN_PLAN` and `HARNESS_BRAIN_MODE` lines (post-merge: the operator's next Finder launch).
- [ ] No bearer reaches the file. A red-first unit arm writes lines carrying the fleet bearer and the plane bearer, and asserts that the file holds only the redaction marker for them.
- [ ] The log rotates once past its cap and never grows unbounded (unit arm).
- [ ] A write failure (read-only dir, full disk) never breaks a boot (unit arm).

## Out of Scope

- Diagnosing or fixing today's own-mode `boot-not-ready`. That follows from the first log a failing launch leaves.
- A log viewer in the cockpit.
- The packaged smoke's own secret census, which is #214.

## Avoided Traps

- **Logging only in smoke.** The smoke already has its isolation root and verdict; the product launch is the blind one.
- **A redaction rule per call site.** One predicate at the file boundary covers every line, including the ones nobody wrote yet.
- **A log under `userData`.** `~/Library/Logs` is where macOS users and Console.app look, and it is not part of the app's state.

## Related

#12 (the shell UX spec: first run) · #7 (the Electron shell epic) · #211 / #212 (the plane record, and the second secret) · #214 (the smoke's census, which can reuse this predicate)

Sweeps: the latest open issues were checked at 2026-09-25 20:47Z (16 open), with no equivalent. A keyword search on "log main process harness persist" found nothing. A2A claims in the last 60 min, all read-states: none on a harness log. Memory Core rationale sweep: recall timed out during tonight's #506 incident (mc-server CPU-starved), so it is attested as unavailable. Own open assignments: none.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c
Retrieval Hint: "packaged shell main process log persisted boot-not-ready Finder launch stdout lost redaction plane bearer"


## Timeline

- 2026-09-25T20:49:32Z @neo-opus-ada added the `enhancement` label
- 2026-09-25T20:49:32Z @neo-opus-ada added the `ai` label
- 2026-09-25T20:49:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T20:49:42Z @neo-opus-ada added parent issue #7
- 2026-09-25T20:52:35Z @neo-opus-ada cross-referenced by PR #220
- 2026-09-25T20:55:00Z @neo-opus-ada cross-referenced by #221
- 2026-09-25T21:32:45Z @neo-opus-ada cross-referenced by #223
- 2026-09-25T22:08:39Z @neo-opus-ada cross-referenced by #225
- 2026-09-25T22:34:09Z @tobiu referenced in commit `4ce50eb` - "Merge pull request #220 from neomjs/ada/219-main-log

feat(harness): the shell keeps its own log under ~/Library/Logs (#219)"
- 2026-09-25T22:34:09Z @tobiu closed this issue

