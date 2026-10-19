---
id: 1203
title: Fleet replaces stale Neural Link Bridge listeners on generation/key mismatch and supervises mid-session exits
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T15:45:11Z'
updatedAt: '2026-10-19T18:34:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1203'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-19T18:34:47Z'
---
# Fleet replaces stale Neural Link Bridge listeners on generation/key mismatch and supervises mid-session exits

## Context

In `neomjs/neo-agent-brain#1144#issuecomment-6256544256` and PR `neomjs/neo-agent-brain#1193` (`pullrequestreview-5574021835` `RA-3`), the operator and FM design seat (`@neo-fable-clio`) locked the conditions for Fleet-managed Neural Link Bridge lifecycle under `neomjs/neo-agent-institution#732` (`L5`):

1. **Supervise, never child:** An FM quit or update must not take the Bridge down with it. Adopt if running and verified, start if absent, and replace at a generation switch or verifier key mismatch.
2. **Keys from custody:** The Bridge's verify key comes from `FleetRegistryService` custody (`signing.key`), never from a static plist.
3. **Coexistence with packaged `own-mode` and `neomjs/neo-agent-institution#753` rehearsal:** In packaged `own-mode` (`harness/main.mjs:1004-1015`) and smoke/rehearsal (`harness/brain.mjs:706,785,1591`), the shell already runs `ORCHESTRATOR_ENTRY` on `NEO_AI_ORCHESTRATOR_DIR`, sets `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED=false`, and `stopRunBridge` (`#753`) tears down detached `run-bridge.mjs` listeners on the run's allocated Bridge port.

## The Problem

PR `#1193` (and forward-fix `#1209`) landed cold-start provisioning and matching-key adoption in `ai/services/fleet/settleFleetBridge.mjs`, leaving two concrete lifecycle gaps:

1. **No replacement on generation or verifier key mismatch (`refused`):** When `probeBridgeVerifier` returns `verdict: 'refused'` (`bridge-verifier-unsupported` from an older generation, `bridge-verifier-unconfigured`, or `bridge-verifier-mismatch` from a prior key), `settleFleetBridge.mjs` immediately returns `bridge-outdated` or `bridge-unkeyed` instead of replacing a stale `run-bridge.mjs` listener on `port` with the current runtime generation's `run-bridge.mjs` and registry key.
2. **No mid-session crash supervision:** Once `settleFleetBridge` settles at `devFleetServer` boot, a detached `run-bridge.mjs` that exits mid-session stays down until the next Fleet server restart.

## The Architectural Reality

- `Prescription checked: ai/services/fleet/settleFleetBridge.mjs — better owner: ai/services/fleet/settleFleetBridge.mjs + ai/services/fleet/devFleetServer.mjs` (Stage 2 intake verification: spawning `ai/daemons/orchestrator/hostEdge.mjs` from `settleFleetBridge` would collide with `own-mode` `Orchestrator` `enforceSingleton()` on `NEO_AI_ORCHESTRATOR_DIR`, fail to start the Bridge under `buildPackagedBrainEnv` / `buildBrainProfile` where `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED='false'`, and break `neomjs/neo-agent-institution#753`'s `stopRunBridge` teardown).
- `ai/services/fleet/settleFleetBridge.mjs` owns Bridge probe, adoption, detached `run-bridge.mjs` launch, and state projection (`ready`, `bridge-outdated`, `bridge-unkeyed`, `bridge-unavailable`).
- `ai/services/fleet/devFleetServer.mjs` owns the Fleet server lifecycle (`startFleetBridgeServer` -> `settleFleetBridge` -> `cleanShutdown`).

## The Fix

1. **Verified Fleet-owned `run-bridge.mjs` replacement on `refused` (`ai/services/fleet/settleFleetBridge.mjs`):**
   - Record the PID of any `run-bridge.mjs` started by Fleet under `<dataDir>/neural-link-bridge-owner.json` (`writeFleetBridgeOwnerRecord` / `readFleetBridgeOwnerRecord`).
   - When `initialProbe?.verdict === 'refused'` and `startIfAbsent !== false` (and `replaceIfRefused !== false`), inspect listeners on `port` (via injectable `listPortListenersFn`, `processCommandFn`, `processParentPidFn`, `killProcessFn`).
   - Only terminate a listener when it matches the Fleet-recorded owner PID in `<dataDir>/neural-link-bridge-owner.json`, its command matches `run-bridge.mjs`, and its parent PID (`ppid`) is `1` (`launchd`/`init`) or the current Fleet process (never signaling foreign processes or a `run-bridge.mjs` supervised by another parent such as `hostEdge.mjs`). Wait briefly for the port to release, then spawn the current runtime generation's `run-bridge.mjs` with `NEO_FLEET_BRIDGE_PUBLIC_KEY` from `FleetRegistryService` custody and poll `probeFn`.
   - When the replacement proves `verdict === 'proved'`, return `{ state: 'ready', verdict: 'proved', action: 'replaced', ... }`. If the listener is foreign or owned by another supervisor, preserve the `bridge-outdated` / `bridge-unkeyed` diagnostic state.
2. **Mid-session Bridge supervision (`superviseFleetBridge` in `ai/services/fleet/settleFleetBridge.mjs`, wired in `ai/services/fleet/devFleetServer.mjs`):**
   - Export `superviseFleetBridge({ port, registry, intervalMs, initialSettlement, settleFn, onStateChange })` which periodically runs `settleFleetBridge` (coalescing in-flight checks and logging only state transitions via `onStateChange`) so a Bridge that exits or loses its key mid-session is automatically restarted/replaced.
   - On `devFleetServer` shutdown (`cleanShutdown`), call `bridgeSupervisor.stop()` to clear the supervision timer without killing the detached `run-bridge.mjs` (**Supervise, never child**).

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `settleFleetBridge` (`ai/services/fleet/settleFleetBridge.mjs`) | `neomjs/neo-agent-brain#1144#issuecomment-6256544256`, `neomjs/neo-agent-institution#753` | Adopts `proved` Bridge (`action: 'adopted'`), starts when absent and records PID in `<dataDir>/neural-link-bridge-owner.json` (`action: 'started'`), replaces `run-bridge.mjs` listener on `refused` only when Fleet ownership is proved (`ownerRecord.pid === listenerPid`, `isBridgeListenerCommand`, and `ppid === 1 \|\| ppid === process.pid`) (`action: 'replaced'`) | Preserves `bridge-outdated` / `bridge-unkeyed` when listener is foreign, supervised by another process (`hostEdge.mjs`), unrecorded, or replacement fails to prove | JSDoc on `settleFleetBridge` & `describeFleetBridgeState` | Unit & loopback process spec (`FleetBridgeKeyProvisioning.spec.mjs`) |
| `superviseFleetBridge` (`ai/services/fleet/settleFleetBridge.mjs`, `ai/services/fleet/devFleetServer.mjs`) | `neomjs/neo-agent-brain#1144#issuecomment-6256544256` Condition 1 (`Supervise, never child`) | Periodic non-overlapping `settleFleetBridge` supervision loop while Fleet runs (seeded with `initialSettlement` and logged on state change via `onStateChange`); `stop()` clears timer on Fleet shutdown without killing detached Bridge | Single boot settle when supervision interval disabled (`<= 0`) | JSDoc | Unit & loopback process spec |

**Decision Record impact:** `none` (refines Fleet Bridge lifecycle within `ai/services/fleet/settleFleetBridge.mjs` and `devFleetServer.mjs`).

## Acceptance Criteria

- [ ] AC-1: Cold start & matching adoption — Fleet boot starts `run-bridge.mjs` detached with `NEO_FLEET_BRIDGE_PUBLIC_KEY` when absent (`action: 'started'`, recording the spawned PID under `<dataDir>/neural-link-bridge-owner.json`) and adopts an already-running `proved` Bridge (`action: 'adopted'`).
- [ ] AC-2: Generation-switch / verifier-mismatch replacement — When `probeBridgeVerifier` returns `refused` (`bridge-verifier-unsupported`, `bridge-verifier-unconfigured`, or `bridge-verifier-mismatch`) and the listener on `port` is a Fleet-owned `run-bridge.mjs` (matching `<dataDir>/neural-link-bridge-owner.json` PID, `run-bridge.mjs` command, and `ppid === 1 || ppid === process.pid`), `settleFleetBridge` terminates the stale listener, starts the current generation's `run-bridge.mjs` with the registry's public key, and settles `{ state: 'ready', verdict: 'proved', action: 'replaced' }` (never signaling a foreign process or a `run-bridge.mjs` owned by another supervisor such as `hostEdge.mjs`).
- [ ] AC-3: Mid-session supervision & FM quit survivability (`superviseFleetBridge`) — `superviseFleetBridge` restarts a Bridge that exits mid-session, logs state transitions via `onStateChange` without repeating unchanged non-ready states every tick, and calling `stop()` on Fleet shutdown leaves the detached Bridge running (**Supervise, never child**).
- [ ] AC-4: Unit tests in `test/playwright/unit/ai/services/fleet/FleetBridgeKeyProvisioning.spec.mjs` cover cold start, matching adoption, generation/key-mismatch replacement (including foreign-listener and supervised/unrecorded `run-bridge.mjs` safety guards), mid-session crash restart, and Fleet shutdown survivability.

## Out of Scope

- Unloading/deleting existing macOS `launchd` plists during FM bootstrap (`neomjs/neo-agent-institution` L5 sibling leaf per `#1144#issuecomment-6256544256`).

## Avoided Traps

- **Spawning `ai/daemons/orchestrator/hostEdge.mjs` from `settleFleetBridge.mjs`:** Collides with `own-mode` `Orchestrator` `enforceSingleton()` on `NEO_AI_ORCHESTRATOR_DIR`, fails to start the Bridge under `buildPackagedBrainEnv` / `buildBrainProfile` where `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED='false'`, and breaks `neomjs/neo-agent-institution#753`'s `stopRunBridge` teardown.
- **Killing foreign listeners on `port`:** Replacement must verify the listener's command matches `run-bridge.mjs` before signaling; a foreign process on `port` is never killed.

## Related

- Successor to `neomjs/neo-agent-brain#1144` / PR `neomjs/neo-agent-brain#1193` (`pullrequestreview-5574021835` `RA-3`) and `neomjs/neo-agent-brain#1209`.
- Specified in `neomjs/neo-agent-brain#1144#issuecomment-6256544256` (`@neo-fable-clio`).
- Coexists with `neomjs/neo-agent-institution#753` (`@neo-opus-ada`).
- Parent goal board: `neomjs/neo-agent-institution#732` (`L5`), rehearsed by `neomjs/neo-agent-institution#736`.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: `query_raw_memories("settleFleetBridge superviseFleetBridge replaced run-bridge 1144 1193 1203")`



## Timeline

- 2026-10-19T15:45:13Z @neo-gemini-julia added the `enhancement` label
- 2026-10-19T15:45:13Z @neo-gemini-julia added the `ai` label
- 2026-10-19T15:45:14Z @neo-gemini-julia added the `architecture` label
- 2026-10-19T15:54:32Z @neo-gemini-julia cross-referenced by PR #1193
- 2026-10-19T16:31:22Z @neo-opus-ada cross-referenced by #752
- 2026-10-19T16:42:59Z @neo-opus-ada cross-referenced by #1208
- 2026-10-19T16:56:45Z @neo-gemini-julia cross-referenced by PR #1209
- 2026-10-19T17:23:32Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T17:36:53Z @neo-gemini-julia changed title from **Fleet supervises the host-edge role across continuity handoffs** to **Fleet replaces stale Neural Link Bridge listeners on generation/key mismatch and supervises mid-session exits**
- 2026-10-19T17:44:50Z @neo-gemini-julia cross-referenced by PR #1212
- 2026-10-19T17:56:30Z @neo-gemini-julia added parent issue #732
- 2026-10-19T18:16:50Z @neo-gemini-julia referenced in commit `09ff913` - "fix(fleet): prove Fleet ownership before signaling Bridge and log state changes (#1203)"
- 2026-10-19T18:34:04Z @tobiu referenced in commit `2b0e2cc` - "feat(fleet): replace stale bridge listeners and supervise mid-session exits (#1203) (#1212)

* feat(fleet): replace stale bridge listeners and supervise mid-session exits (#1203)

* fix(fleet): prove Fleet ownership before signaling Bridge and log state changes (#1203)"
- 2026-10-19T18:34:47Z @tobiu closed this issue
- 2026-10-19T18:45:57Z @neo-opus-ada cross-referenced by #732
- 2026-10-19T19:27:38Z @neo-gemini-julia cross-referenced by #1215

