---
id: 1183
title: 'A recovered Codex Desktop seat reads its saved app path as an array, so its helper cleanup never runs'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T11:48:13Z'
updatedAt: '2026-10-19T12:36:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1183'
author: neo-opus-vega
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
closedAt: '2026-10-19T12:00:42Z'
---
# A recovered Codex Desktop seat reads its saved app path as an array, so its helper cleanup never runs

## Context

On 2026-10-19 at 11:40:57Z, Emmy's Start on the FM with #1172 (Brain c25e1015) still refused: admission was reserved and revoked `start-failed` 0.85 s later, before any spawn. Maryam found the cause (A2A 11:47Z): the recovery #1172 added reads the saved Start's `launchCommand` as an array.

## The Problem

`recoverRuntimeCustodyUnderGuard` (`FleetLifecycleService.mjs` ≈:2383) sets `record.launchCommand ??= Array.isArray(start.launchCommand) ? start.launchCommand[0] : null`. The seat runtime record stores `launchCommand` as one absolute path (`seatRuntimeRecord.mjs`: `isSeatRuntimeRecord` requires `absolute(value.launchCommand)`; `createSeatRuntimeRecord` copies `record.launchCommand`). So the recovered record gets `null`, `probeCrashpadExecutable(null)` finds no helper, and `start()` refuses with the helper-cleanup refusal. #1172's arm passed because its hand-made saved Start used an array, a shape the record's codec never produces.

## The Fix

Take `start.launchCommand` as the codec gives it: the absolute path. The arm builds its saved Start in the codec's shape and checks it with `isSeatRuntimeRecord`'s path rule, so the fixture cannot drift from the record again.

## Acceptance Criteria

- [ ] A recovered codex-desktop record owing cleanup carries the saved app path; its next Start runs the helper cleanup and spawns.
- [ ] The arm's saved Start uses the codec's shape (a string `launchCommand`); with the old array read, it fails.
- [ ] [L3-deferred — operator handoff needed] Installed: Emmy starts on the next FM candidate. Residual-Owner: neomjs/neo-agent-institution#736.

Related: #1169, #1172, neomjs/neo-agent-skills#191 (tests derive from the module's contract).


## Timeline

- 2026-10-19T11:48:15Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T11:48:15Z @neo-opus-vega added the `bug` label
- 2026-10-19T11:48:15Z @neo-opus-vega added the `ai` label
- 2026-10-19T11:49:37Z @neo-opus-vega cross-referenced by PR #1184
- 2026-10-19T12:00:42Z @tobiu referenced in commit `99804a5` - "fix(fleet): a recovered Codex Desktop seat keeps its saved app path for the helper cleanup (#1183) (#1184)

The recovery read the saved Start's launchCommand as an array, but the seat
runtime record stores one absolute path, so the record got null, the crashpad
probe found nothing, and the seat's Start refused. The arm now builds its saved
Start in the record's own shape, so the fixture cannot drift from the codec."
- 2026-10-19T12:00:42Z @tobiu closed this issue
### @neo-opus-vega - 2026-10-19T12:36:44Z

**Installed receipt (L3 AC): Emmy starts on the next FM candidate.** ✅

- **FM:** Brain `99804a5` (this fix), Engine `1be83e0`, Institution `e2e10e0`. Staged 12:07:53Z, installed 12:11Z, no harness closed.
- **Before the Start:** her recovered row read `failed` with ownership and cleanup owed. The saved `launchCommand` was a string (the codec's shape), `profileArg` was present, and the saved pid was gone.
- **Start:** reserved 12:35:03Z, generation active 12:35:09.777Z. The row now reads `running` with `ownershipUnresolved: false` and `cleanupUnresolved: false`, so the owed helper cleanup ran and the seat spawned. 17 processes run under her profile.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-10-19T12:36:56Z @neo-opus-vega cross-referenced by #1169
- 2026-10-19T12:36:57Z @neo-opus-vega cross-referenced by #1178

