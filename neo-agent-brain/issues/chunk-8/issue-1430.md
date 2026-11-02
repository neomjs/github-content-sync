---
id: 1430
title: No daemon diagnoses the plane's Docker disk or prunes its build cache
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-02T05:09:57Z'
updatedAt: '2026-11-02T17:12:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1430'
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
closedAt: '2026-11-02T14:42:29Z'
---
# No daemon diagnoses the plane's Docker disk or prunes its build cache

## Context

On 2026-11-02 the local plane's Docker data disk (colima `vz` guest, engine 29.2.1, API 1.53), `/dev/vdb1`, measured **97 % used: 147.1 GiB total, 5.6 GiB free**. The number has not moved since 02:55Z. `docker system df` breaks it down:

| class | size | reclaimable |
|---|---:|---:|
| images | 94.2 GB | 30.9 GB, all tagged (0 dangling) |
| volumes | 61.6 GB | 31.9 GB, unattached |
| build cache | 17.1 GB | 14.0 GB (0 of 134 entries active) |

Chroma's 17 GB volume and the shared SQLite volume sit on this disk, so when the disk fills, every seat's store writes fail. The likely trigger is the next plane rebuild, whose new layers and build cache land in the 5.6 GiB that remains (an inference, not observed). Promoted from defect-note `cae246a4` after the disk-hygiene owner (@neo-opus-ada) confirmed the Nov 1 round covered the host volume only.

## The Problem

The self-healing daemons watch containers, never the disk beneath them. ADR 0025 §2.4's detect signals are per container (memory, CPU, health, config-correctness, restart churn, data-integrity), and no target in ADR 0026 §2.4's action matrix can return disk. The only remedy today is someone noticing and pruning by hand, which is the operator-pointed repair ADR 0025/0026 exist to retire.

Exactly one class is safe to reclaim autonomously: **build cache**. The next build regenerates it, it holds no state, and no running container depends on it. The unused images are all tagged here, so any of them may be a rollback target, and unattached volumes may hold restore data. Both need an owner's read and stay record-only.

## The Architectural Reality

- **The diagnosis needs no new privilege for capacity.** The orchestrator's own data volume lives on the engine's data disk: `fs.statfsSync('/app/.neo-ai-data')` inside the orchestrator container reads 147.1 GiB total and 5.6 GiB free, the same as the guest's `df /var/lib/docker`. Reclaimable sizes come from `GET /system/df`. The read-observe set in `ai/daemons/orchestrator/services/DeploymentRuntimeAccessService.mjs` is `inspect`, `logs`, `stats` and `events`, so this adds one read operation.
- **The actuator needs one new L0 lifecycle operation.** `DEPLOYMENT_RUNTIME_LIFECYCLE_OPERATIONS` (same file, l.17) holds `restart` and `update-memory-limit`. A bounded build-cache prune would be the third: `POST /build/prune` with a reserved warm floor, the engine's `--reserved-space` / `--min-free-space`. ADR 0026 AC-9 forbids widening the closed action set through an implementation sub alone, so the amendment ships in the same change, as §2.8 did for `update-memory-limit`.
- **The target is the engine, not a compose service.** ADR 0026 §2.4 keys actions by target kind: supervised task, compose service (by declared class), deploy target. Build cache belongs to none of them, so this needs a new row.
- **Diagnosis home:** `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs`. Lifecycle-write exposure stays in `ai/daemons/orchestrator/control-plane` (ADR 0026 §2.7). Host-side precedent: `ai/services/fleet/probePlacement.mjs` reads the host's root disk (`df -k /`) for seat placement. Nothing reads the plane's disk.

## The Fix

1. **Detect** (ADR 0025 §2.4 amendment): a `disk-pressure` fact from `statfs` on the orchestrator's data mount, sampled over a window like the memory facts, carrying the `/system/df` class breakdown.
2. **Diagnose**:
   - sustained pressure with reclaimable build cache above a floor → `prune-build-cache`;
   - pressure without reclaimable build cache → `record` with the breakdown. Images and volumes are named, never acted on.
3. **Act** (ADR 0026 amendment): an **engine** target kind that admits exactly `prune-build-cache`, implemented as the L0 operation with a warm floor so the next build still hits cache. It runs inside the §2.5 envelope: durable `heal_attempts`, a token bucket, and alarm-only when a prune frees nothing.
4. **Receipt**: free bytes before and after, read by the same `statfs`.

## Contract Ledger

*Backfilled 2026-11-02 by the ticket author during the PR #1431 review (pr-review §5.4). Existing surfaces were read at `1d5c2757` and against the live engine (29.2.1, API 1.53).*

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `orchestrator.diskPressure.{percent, windowMs, minReclaimableBuildCacheBytes, buildCacheWarmFloorBytes}` (new leaves) | this ticket; ADR 0019 §5.1 | Declared once in `ai/configBase.mjs`. The diagnosis reads the three thresholds; the warm floor is read by the actuator alone, at its use site. | Env-overridable per leaf; no second default anywhere else | JSDoc | `config-leaf-parity.json` + AiConfig lints |
| `orchestrator.deploymentRuntimeAccess.readOperations` / `lifecycleOperations` defaults (existing leaves) | ADR 0026 §2.6 AC-9 + AC-14 | Gain `system-df` / `prune-build-cache`; the closed L0 sets gain the same | An allowlist without them disables the lane: no read, no prune | ADR 0026 §2.9 | `DeploymentRuntimeAccessService.spec.mjs` |
| L0 `readObserve({operation: 'system-df'})` (new) | ADR 0026 §2.9 | `statfs` on the resolved `AiConfig.plane.dataRoot`, plus `GET /system/df` normalized per class. A build-cache record counts as reclaimable only when it is neither `InUse` nor `Shared`. | A statfs or socket failure is a `readFailed` advisory fact, never a diagnosis | JSDoc | Spec on an API-1.53-shaped fixture with `Shared: true` records |
| L0 `applyLifecycle({operation: 'prune-build-cache', keepStorageBytes})` (new) | ADR 0026 §2.9 + AC-14 | Refuses a non-positive-integer floor before the socket. Sends `POST /build/prune` carrying the floor, serialized per engine. Receipt: `statfsBefore`, `statfsAfter`, `spaceReclaimedBytes`, `cachesDeleted`. | No image, volume, container or system prune path is reachable | ADR 0026 §2.9 | AC-3 / AC-4 specs |
| `RecoveryActuatorService.apply('docker-engine', 'prune-build-cache')` (new matrix row) | ADR 0026 §2.4 | The actuator resolves the warm floor itself; no caller supplies bytes. A zero reclaim yields `recorded` / `prune-freed-nothing`, alarm-only for the current attempt window. | `blockedEngines` removes the target | ADR 0026 §2.4 | `RecoveryActuatorService.spec.mjs` |
| `orchestrator.recoveryActuator.blockedEngines` (new leaf) | ADR 0026 §2.4 | Blocklist for engine targets | `[]` | JSDoc | `RecoveryActuatorService.spec.mjs` |
| Deployment-state snapshot section `engineDisk` (new, additive) | ADR 0025 §2.4 | `{recordType: 'deployment-engine-disk-state', targetIdentity: {kind: 'engine', id: 'docker-engine'}, status, statfs, df, diagnosis, proof, errors}` | Snapshots without the section stay valid | — | Bridge spec + `offHostSync.spec.mjs` |

## Acceptance Criteria

- [ ] The PR that ships the code also amends ADR 0025 §2.4 (the disk-pressure signal) and ADR 0026 §2.4/§2.6 (the engine row and the AC-9 widening).
- [ ] A spec drives pressure plus reclaimable cache to exactly one bounded `prune-build-cache`, and pressure without reclaimable cache to `record` only.
- [ ] A negative spec proves that no image or volume operation is reachable.
- [ ] The L0 boundary refuses a prune that lacks the warm floor, and the operation is serialized per engine.
- [ ] Post-merge, on this plane: the receipt shows `statfs` free bytes before and after the first autonomous prune. `[L4-deferred — post-merge receipt; Residual-Owner: #1416]`

## Out of Scope

- Pruning images or volumes: it needs an owner's read, so here they are record-only.
- Resizing the colima disk.
- Host-volume hygiene (the Nov 1 round).
- Chroma's own growth (#60).

## Avoided Traps

- **A one-off manual prune as the fix.** It resets the disk and leaves the gap open for the next fill.
- **`docker system prune`.** It reaches images and volumes.
- **A new mount of `/var/lib/docker`.** The orchestrator's own volume already reports the same filesystem.

## Decision Record impact

Amends ADR 0025 §2.4 (detect) and ADR 0026 §2.4/§2.6 (action matrix, AC-9 widening).

## Related

#1416 (resident memory; disk was out of its scope) · #60 (Chroma growth) · defect-note `cae246a4`.

Live latest-open + closed sweep: checked the latest 20 open and closed searches ("docker disk prune build cache", "disk pressure") at 2026-11-02 05:12Z. Same-surface hits (#1416, #60, #66, closed #685 on placement RAM budgets) were read; no equivalent or prior exclusion was found. A2A claim sweep: the last 30 messages in all read states show no disk or prune claim. Memory Core rationale sweep: only the Oct 22 Docker measurement (`09f8375d`) and the Nov 1 host round. Own-assignment sweep: 12 open, none on disk. Structure map: owning folders are `ai/daemons/orchestrator/services` and `control-plane`.

unowned-rationale: needs a live plane plus an ADR 0025/0026 amendment, so it is held for a Brain seat. @neo-opus-vega (plane owner) reviews.

Origin Session ID: 2199a4ed-a9d8-4714-8a57-a31e4675bdfb
Retrieval Hint: "plane docker disk pressure build cache prune actuator engine target"

Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-11-02T05:09:59Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T05:09:59Z @neo-opus-vega added the `ai` label
- 2026-11-02T05:09:59Z @neo-opus-vega added the `architecture` label
- 2026-11-02T05:09:59Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T05:16:35Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T05:45:36Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-02T06:30:17Z @neo-gemini-julia cross-referenced by PR #1431
- 2026-11-02T06:43:06Z @neo-gemini-julia referenced in commit `1d5c275` - "feat(orchestrator): satisfy ADR-0019 leaf-read lint and additive snapshot section spec (#1430)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>"
- 2026-11-02T09:15:12Z @neo-gemini-julia referenced in commit `d10921a` - "fix(orchestrator): address Round-1 review on plane docker disk pressure (#1430)"
- 2026-11-02T14:42:29Z @tobiu referenced in commit `b07a507` - "feat(orchestrator): diagnose plane docker disk pressure and prune build cache (#1430) (#1431)

* feat(orchestrator): diagnose plane docker disk pressure and prune build cache (#1430)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>

* feat(orchestrator): satisfy ADR-0019 leaf-read lint and additive snapshot section spec (#1430)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>

* fix(orchestrator): address Round-1 review on plane docker disk pressure (#1430)"
- 2026-11-02T14:42:30Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-02T17:12:08Z

**Deployed on the local plane at 17:11Z, 2026-11-02: Brain `59c5a551` (containing #1431 at `b07a5074`), up from `804908b7`.** AC-5's autonomous prune is not due yet; the disk sits below the trigger.

**Cut receipt.** `cut.sh 59c5a551` ran 17:09:22–17:11:05Z.
- **Preflight:** `PROCEED_VERIFIED`. A restorable pre-transition bundle exists (146,454 rows).
- **Services:** mc-server, kb-server, orchestrator and fleet-server were recreated from prebuilt images (build 17:04:44–17:05:14Z). All four were healthy at 17:11:04Z, with the image label and `/app/.neo-revision` both reading `59c5a551`.
- **Untouched:** chroma (`b89d731f60ea`) and ingress (`bf26d90ce88a`) have the same ids before and after.
- **Writable layers:** `docker diff` beforehand showed only the `/app/.neo-ai-data` mountpoint on each container.
- **Wake and host-edge daemons:** booted out and back in, both exit 0, both running.
- **Memory Core** reports `deployedRevision 59c5a551`.
- **Allowlists:** no env-file or compose override pins `NEO_ORCHESTRATOR_RUNTIME_ACCESS_{READ,LIFECYCLE}_OPERATIONS`, so the defaults with `system-df` and `prune-build-cache` apply.

**AC-5 baseline, guest `statfs` at 17:11:28Z (after the cut).**

| field | value |
|---|---|
| blocks × bsize | 38,557,897 × 4,096 = 157,933,146,112 B |
| `bfree` | 27,406,811,136 B |
| `bavail` | 19,874,041,856 B |
| build cache | 5.296 GB, of which 2.208 GB is reclaimable |

The lane computes `freeBytes` from `bfree` (`DeploymentRuntimeAccessService.mjs` l.1464), so it reads the disk at **82.65 % used**. The trigger is ≥ 90 % sustained for 30 s, with at least 1 GiB of reclaimable cache. So no autonomous prune is due, and none should be. The cut itself freed about 2.8 GB: `bavail` went from 17,040,080,896 B at 17:05:41Z to 19,874,041,856 B.

The lane writes nothing below the threshold, so its liveness at 82.65 % is unobserved. AC-5's before/after receipt stays open under #1416 until the disk first crosses 90 %.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-11-02T17:34:39Z @neo-opus-vega cross-referenced by #1433

