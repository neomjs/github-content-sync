---
id: 581
title: 'The bundled Fleet reads its local snapshot path even when attached to a plane, so FM''s System view says "snapshot missing" while the plane''s snapshot is one existing call away'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T14:01:47Z'
updatedAt: '2026-09-27T14:40:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/581'
author: neo-opus-vega
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
closedAt: '2026-09-27T14:40:20Z'
---
# The bundled Fleet reads its local snapshot path even when attached to a plane, so FM's System view says "snapshot missing" while the plane's snapshot is one existing call away

## Context

Operator release goal, 2026-09-27: "Getting FM connected again to our local Agent OS." The installed Fleet Manager (Institution `4ca542f`, bundled Brain `c6c92c2`; receipt neomjs/neo-agent-institution#7 comment 5855775306, @neo-gpt-emmy) attaches to the local plane at 127.0.0.1:3102 and shows real A2A, but its System keeper-view reads `snapshot-missing`. On the same plane at the same time, `get_deployment_state_snapshot` answers `{ok: true, status: 'available', ageMs: 37856, services: 5}` (Emmy's live probe 13:5xZ; mine at 13:08Z read the same shape with `snapshot.services` of five). Handed to me by Emmy 13:56Z with the wire named; no new design.

## The Problem

`ai/services/fleet/devFleetServer.mjs:151–164`: the fleet-server entry wires `FleetControlBridge.deploymentStateSource` through `wireDeploymentStateReadSource({path: AiConfig.orchestrator.deploymentStateBridge.snapshotPath, …})` UNCONDITIONALLY — the local snapshot file of this process's own data root. In plane mode (`fleet.planeBase` set, `planeClient` admitted) every other truth seam already reads the plane: mailbox, compose, catch-up, presence (`createPlaneWhoIsOnlineReader`), wake identities and observations. The deployment state does not, and the installed app's own data root has no orchestrator writing that file, so `fleetDeploymentState()` projects `unavailable` and the System view renders `snapshot-missing` for a plane whose snapshot is available.

## The Architectural Reality

- `createDeploymentStateReadSource({path, staleAfterMs, maxBytes, readImpl, now})` already takes an injectable `readImpl` and maps the READER'S VERDICT (`{status, snapshot, ageMs, reason}`) onto the closed wire projection via `projectDeploymentStateForFleet` — the redaction Emmy asked to preserve lives there, not in the reader.
- The plane tool `get_deployment_state_snapshot` IS `readDeploymentStateSnapshot` served over MCP: its answer carries the same verdict fields (`ok, status, filePath, ageMs, staleAfterMs, snapshot, schemaDiagnostics, reason`), so a plane reader is one adapter with no re-interpretation.
- `planeMailboxClient.callTool(name, args)` returns the mapped structured payload (as `planeWhoIsOnlineReader` consumes it) and throws on tool-level errors.
- `wireDeploymentStateReadSource` leaves the seam unwired on an empty path — correct for the file reader, wrong for a plane reader that needs no path.
- #314 (closed) built the wire verb over the local file; neomjs/neo-agent-brain#459 (readers' `resources/content` fallbacks) is a separate, broader boundary and does not gate this.
- Structure map: `ai/services/fleet` — the new reader sits beside `planeWhoIsOnlineReader.mjs` / `planeWakeIdentitiesReader.mjs` (the same plane-reader shape), specs beside theirs.

## The Fix

1. `ai/services/fleet/planeDeploymentStateReader.mjs`: `createPlaneDeploymentStateReader(planeClient)` returns a `readImpl` that calls `get_deployment_state_snapshot` through the admitted plane client and hands the verdict back untouched (an answer without a string `status` throws, like its siblings).
2. `wireDeploymentStateReadSource` accepts `readImpl`; with one supplied, an empty path no longer leaves the seam unwired.
3. `devFleetServer.mjs`: plane mode wires the plane reader (boot log names the plane); in-process mode keeps the file reader.
4. Arms: the plane reader hands the plane's verdict to the source and the projection reads `available` with the plane's services; an unreadable answer projects `unavailable` with a reason; wiring with a reader and no path wires (red on dev: `null`).

## Acceptance Criteria

- [ ] AC-1: in plane mode `fleetDeploymentState()` serves the plane's snapshot projection (`state: 'available'`, the plane's services) with `projectDeploymentStateForFleet`'s redaction unchanged; in-process mode is unchanged (unit arms, red-first).
- [ ] AC-2: a plane answer without a snapshot projects `unavailable` with the plane's reason, never a fabricated plane (unit arm).
- [ ] AC-3 (L4, on neomjs/neo-agent-institution#7): the installed FM, attached to the local plane, shows the System view populated — Emmy / root verify after the Brain pin bump and repackage.

## Out of Scope

- The PR/lane corpus fallback (#459) and mc-server's corpus mount (#64 AC-8).
- The empty fleet registry on the plane (no agents defined; each needs a PAT under the operator's ruling) — the roster stays empty until agents are added, a data step, not a wire.

## Related

#314, #459, neomjs/neo-agent-institution#7, neomjs/neo-agent-institution#12

Live latest-open sweep at 2026-09-27T14:00Z: no open Brain issue on the bundled fleet's deployment-state source (searches `deployment-state`, `fleet plane snapshot`); #314 closed. A2A: Emmy's 13:56Z handoff names me as owner; no competing claim. Structure map: `ai/services/fleet`, one new reader file beside its siblings.

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a

## Timeline

- 2026-09-27T14:01:47Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T14:01:49Z @neo-opus-vega added the `bug` label
- 2026-09-27T14:01:49Z @neo-opus-vega added the `ai` label
- 2026-09-27T14:01:49Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T14:06:21Z @neo-opus-vega cross-referenced by PR #582
- 2026-09-27T14:07:02Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T14:08:36Z @neo-preview cross-referenced by PR #577
- 2026-09-27T14:31:09Z @neo-opus-vega referenced in commit `941232e` - "fix(fleet): the FM projection carries the reader's own freshness verdict — a plane's stale picture never reads ok under the local horizon (#581)

The projection re-aged every snapshot against the bundled Fleet's staleAfterMs leaf and ignored
the reader's status, so a plane verdict decided on the plane's horizon could flip either way, and
a horizon of 0 (staleness disabled) read every snapshot stale — on the local file path too.
projectDeploymentStateForFleet now takes the verdict (`stale`) instead of a horizon; the reader
is the one authority. Four arms red at e0077c5: plane stale under a longer local horizon, plane
available under a shorter one, plane horizon 0, local horizon 0."
- 2026-09-27T14:40:21Z @tobiu referenced in commit `a7aee92` - "Merge pull request #582 from neomjs/vega/581-plane-deployment-state

fix(fleet): a fleet process attached to a plane reads the plane's deployment snapshot, not its own empty data root — FM's System view stops saying snapshot-missing (#581)"
- 2026-09-27T14:40:21Z @tobiu closed this issue
- 2026-09-27T14:52:45Z @neo-opus-vega cross-referenced by #585
- 2026-09-27T15:09:54Z @neo-opus-vega cross-referenced by PR #588

