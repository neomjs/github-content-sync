---
id: 214
title: The packaged smoke proves a stored-plane boot against a fixture plane
state: OPEN
labels:
  - enhancement
  - ai
  - testing
assignees: []
createdAt: '2026-09-25T17:02:02Z'
updatedAt: '2026-09-25T22:26:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/214'
author: neo-opus-ada
commentsCount: 0
parentIssue: 12
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
# The packaged smoke proves a stored-plane boot against a fixture plane

## Context

#211 gives the packaged shell its own plane record: `plane.json` plus a `safeStorage`-encrypted bearer under `userData`, read at boot into the fleet child's env (`planeEnvFragment` in the packaged env, `harness/main.mjs`). PR #212 pins every seam at unit level. On 2026-09-25 the lead split the end-to-end smoke arm out of #211 (see Fix 3 in its body) for three reasons: no CI job runs the harness smoke (`.github/workflows/` has none), a packaged run would write a `safeStorage` key into the runner's login keychain on the shared team machine, and the seams are already pinned. This leaf is that arm.

## The Problem

No automated run proves the composition the unit arms cannot. Nothing checks that a stored record becomes a `plane-attach` boot, that the fleet child is admitted by a plane, or that a roster reaches the renderer. The smoke's secret census is also blind to the new secret: it watches only `fleetBearerToken`, so a plane bearer in a Brain log line, a renderer error, or an IPC reply would pass silently.

## The Architectural Reality

- **Isolation.** Both smoke profiles force the plane leaves empty: `NEO_FLEET_PLANE_BASE`/`NEO_FLEET_PLANE_BEARER` are `''` in `buildBrainProfile` (`harness/brain.mjs`, checkout) and in `bootSmokeBrain`'s packaged profile (`harness/main.mjs`). `harness/README.md` states that machine-level exports cannot redirect the smoke's Fleet process into the canonical plane. A fixture arm has to open that door on purpose, and only to a loopback plane it owns.
- **Admission.** In plane mode the fleet child (Brain `ai/services/fleet/devFleetServer.mjs`) admits the viewer against `<planeBase>/mc/mcp` through `createPlaneMailboxClient`. It then logs either `mailbox/compose/catch-up seams bound to the containerized plane at …` or `plane mode refused (…)`. Those two lines are the arm's observable.
- **Census.** `recordSmokeFailure`, `brainLog`, and the headed smoke's IPC reply census all compare against `fleetBearerToken` only (`harness/main.mjs`).
- **Custody.** `readPlaneConfig` / `writePlaneConfig` (`harness/planeConfig.mjs`) take `safeStorage` as a parameter. The record path can therefore run against the smoke's isolation root with an injected encryption stand-in, without touching the OS keychain.

## The Fix

1. **Fixture-plane arm.** Add a flag beside `NEO_HARNESS_SMOKE` that does three things:
   - starts a loopback plane on a runtime-allocated port;
   - writes a record into the smoke's isolation root, never the user's real `userData`;
   - boots the fleet child in `plane-attach` from that record.
   **Decision (2026-09-25, read at Brain `dev` `60807a5`):** use an isolated mc-server under the smoke profile, in `seat-token` auth mode, not a stub.
   - Since #224, the attach needs the bearer's **identity**: the shell's probe and the fleet child's `init` both read `list_permissions`' `identity`.
   - `local-bearer` mode proves possession only (`AuthService.createLocalBearerVerifier` returns no `userId`), so its attach would refuse as `no-identity`.
   - `seat-token` binds a minted subject per request (`AuthService.setupSeatToken`).
   - `mintSeatToken({agentIdentityNodeId})` (`ai/mcp/server/shared/helpers/seatToken.mjs`) mints the fixture's bearer and registry row for a fixture identity seeded in the isolated graph.
   - A stub would re-encode the Streamable-HTTP session flow the probe and the client both use, so it would need its own parity arm.
   - **Precondition:** checkout runs share the installed app's `userData` (`neo-harness`), so the smoke must first get its own `userData`. See the 2026-09-25 defect-note.
2. **No keychain write.** In this mode the record goes through an encryption stand-in bound only to the smoke. Electron's `safeStorage` stays Electron's contract, and the unit arms already fake it.
3. **Census.** The census watches a set of bearers: the fleet bearer plus the plane bearer from the record or `NEO_FLEET_PLANE_BEARER`. Red-first: an injected line carrying the plane bearer must fail the smoke.

## Acceptance Criteria

- [ ] The fixture arm boots `plane-attach` from a record in the isolation root, on a checkout and on a packaged build. Four checks must pass:
  - the boot fact reports `{mode: 'plane-attach', up: true}`;
  - `planeStatus()` answers `configured: true, attached: true`;
  - the admission line names the fixture plane;
  - a renderer `listAgents` round trip is served through it.
- [ ] The run leaves the runner's login keychain unchanged (`security dump-keychain` item count before = after) and writes nothing under the real `userData`.
- [ ] A red-first arm puts the plane bearer into a Brain log line, a renderer error, and an IPC reply; each one fails the smoke with a `secretLeaks` entry. The green run reports none.
- [ ] Without the flag, `npm run smoke` and `npm run smoke:brain` keep both plane leaves empty and stay green.

## Out of Scope

- A CI job for the harness smoke.
- Electron's own keychain encryption.
- The setup card's UI, which #211's unit arms pin.

## Avoided Traps

- **Pointing the arm at the canonical plane.** It breaks the isolation matrix, and the admission would need a real PAT.
- **A real `safeStorage` write on the shared machine.** It lands in whoever's login keychain runs it, which is the reason this arm left #211.
- **A census keyed on one secret.** Every new credential the shell holds joins the set, or the census certifies nothing about it.

## Related

#211 (split from; its PR #212 lands the record and the seams) · #12 (parent) · #7 (the Electron shell epic)

Sweeps: live latest-20 open issues at 2026-09-25 17:00Z, no equivalent (#211 is the source, #11 the visual harness, #14 the TTFP instrument). A2A claims in the last 60 min, all read-states: none on a smoke plane arm. Memory Core: no prior decision on this surface (top hits unrelated). Own open assignments: #211 only.

unowned-rationale: parked outside today's three lanes; the natural pickup is #211's implementer once #212 merges and the lane focus lifts.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c
Retrieval Hint: "harness smoke fixture plane plane-attach safeStorage keychain secret census plane bearer"



## Timeline

- 2026-09-25T17:02:04Z @neo-opus-ada added the `enhancement` label
- 2026-09-25T17:02:05Z @neo-opus-ada added the `ai` label
- 2026-09-25T17:02:05Z @neo-opus-ada added the `testing` label
- 2026-09-25T17:02:10Z @neo-opus-ada added parent issue #12
- 2026-09-25T17:02:11Z @neo-opus-ada cross-referenced by #211
- 2026-09-25T17:03:02Z @neo-opus-ada cross-referenced by PR #212
- 2026-09-25T20:49:32Z @neo-opus-ada cross-referenced by #219
- 2026-09-25T20:55:00Z @neo-opus-ada cross-referenced by #221
- 2026-09-25T21:32:45Z @neo-opus-ada cross-referenced by #223
- 2026-09-25T22:08:39Z @neo-opus-ada cross-referenced by #225

