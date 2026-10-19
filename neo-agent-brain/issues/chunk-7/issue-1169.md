---
id: 1169
title: 'A gone Codex Desktop seat with owed helper cleanup never recovers: ownership waits on cleanup, and cleanup lacks its profile'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-19T02:42:47Z'
updatedAt: '2026-10-19T12:36:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1169'
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
closedAt: '2026-10-19T03:25:27Z'
---
# A gone Codex Desktop seat with owed helper cleanup never recovers: ownership waits on cleanup, and cleanup lacks its profile

## Context

`neo-gpt-emmy` (codex-desktop) was stopped from the FM on 2026-10-18. Its journal row still reads `state: stopped, ownershipUnresolved: true, cleanupUnresolved: true` at issuer epoch 14 on the new FM (Brain e4886d2a). The row made the old FM refuse every continuity handoff on 2026-10-19 (`seat neo-gpt-emmy is not a live original Start`), and it makes the new FM refuse Emmy's Start.

## The Problem

`recoverRuntimeCustodyUnderGuard` (`FleetLifecycleService.mjs` ≈:2336–2351) has two faults:
1. **Ownership waits on cleanup.** It copies `saved.cleanupUnresolved` onto the recovered record and clears `ownershipUnresolved` only when the seat process is gone *and* no helper cleanup is owed. A gone seat therefore stays "alive but cannot be identified", and `start()` refuses it first (≈:800).
2. **The recovered record lacks what cleanup needs.** It carries no `electronProfile` or `launchCommand`, so neither `stop()` nor `start()` can run `finalizeCodexDesktopHelpers`, the only path that clears `cleanupUnresolved`.

The saved Start holds both: `profileArg` `--user-data-dir=<instanceHome>/electron-profile`, and `launchCommand[0]`, the app executable. The row never resolves, across any number of FM restarts.

## The Fix

When the seat process is gone, mark ownership resolved. If helper cleanup is still owed for a `codex-desktop` seat, give the record its profile, from `guiProfileDir` (`deriveHarnessLaunchSpec.mjs`, exported), and its executable, from the saved Start, so the next Stop or Start runs the existing helper cleanup and then proceeds.

## Acceptance Criteria

- [ ] After recovery, a gone codex-desktop seat with owed cleanup is not refused as an unidentified live seat.
- [ ] Its next Start runs the helper cleanup once and spawns; a failing cleanup still refuses with the cleanup reason.
- [ ] A gone seat with no owed cleanup still recovers `stopped` (existing arms unchanged).
- [ ] Installed: Emmy starts on the next FM candidate.

Related: #1074 (helper cleanup), #1143 / #1166 (handoff), #1167 (Codex project check), neomjs/neo-agent-institution#732.


## Timeline

- 2026-10-19T02:42:48Z @neo-opus-vega added the `bug` label
- 2026-10-19T02:42:48Z @neo-opus-vega added the `ai` label
- 2026-10-19T02:57:23Z @neo-opus-vega cross-referenced by #1170
- 2026-10-19T03:08:26Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-19T03:13:28Z @neo-fable-mnemosyne cross-referenced by PR #1171
- 2026-10-19T03:13:35Z @neo-opus-vega cross-referenced by PR #1172
- 2026-10-19T03:25:27Z @tobiu referenced in commit `b55e253` - "fix(fleet): a gone Codex Desktop seat owing helper cleanup recovers resolvable (#1169) (#1172)

Recovery cleared ownership only when the seat was gone and no helper cleanup was
owed, and the rebuilt record lacked the profile and app the cleanup needs, so such
a seat stayed alive-but-unidentified across every FM restart and its Start was
refused. A gone seat's ownership now resolves, and a codex-desktop record owing
cleanup gets its profile (guiProfileDir, now exported) and app from the saved
Start, so the next Start runs the existing helper cleanup and spawns."
- 2026-10-19T03:25:27Z @tobiu closed this issue
- 2026-10-19T10:56:22Z @neo-opus-vega cross-referenced by #1178
- 2026-10-19T11:48:14Z @neo-opus-vega cross-referenced by #1183
### @neo-opus-vega - 2026-10-19T12:36:55Z

**Installed receipt: Emmy starts on the next FM candidate.** ✅

Her row was stopped on 10-18 with ownership and cleanup unresolved. It recovered on FM Brain `99804a5` (with #1184, the array read this recovery first shipped with) and started at 12:35:09Z. The row now reads `running` with both flags cleared: the owed helper cleanup ran once and the seat spawned. Full receipt: https://github.com/neomjs/neo-agent-brain/issues/1183#issuecomment-6267364930

— Vega (Opus 5.5, Claude Code) 🌿



