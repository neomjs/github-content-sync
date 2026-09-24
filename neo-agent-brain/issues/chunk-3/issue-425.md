---
id: 425
title: Local Agent OS writers keep plane state outside every named volume
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-23T11:29:45Z'
updatedAt: '2026-09-24T16:48:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/425'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-09-23T12:55:38Z'
---
# Local Agent OS writers keep plane state outside every named volume

## Context

The #253 cut ([receipt F1](https://github.com/neomjs/neo-agent-brain/issues/253#issuecomment-5793917290)) recreated kb-server, mc-server and orchestrator on the canonical local plane. `docker diff` on the old containers showed state that no named volume covers: orchestrator `concepts/` (136 nodes, 182 edges), `memory-core/lazy-edges.jsonl` (96 lines), `rem-runs/` (200 files), its wake cursor and `.gitmirror-ssh/known_hosts`, plus mc-server's wake cursor. A plain recreate would have dropped all of it. The cut kept it only by copying it out after the graceful stop and back in before the first start.

Re-running `docker diff` on the post-cut containers found a second instance of the same class, and this one is a regression. kb-server and mc-server write their heap-observation records into their own writable layers. The orchestrator's read-only `shared-heap-observation-data` volume still holds `kb-server.json` and `mc-server.json` from **2026-08-25 21:59Z**, four weeks stale.

## The Problem

Two defects, one class: a service writes plane state under `/app/.neo-ai-data` where no volume is mounted.

1. **The heap-observation writers lost their mount.** neomjs/neo#16810 gave the channel a shared mount (neomjs/neo#16839, 2026-08-10). neomjs/neo#17772 (commit `467fd122f3`, 2026-08-25) replaced the `shared-heap-observation-data` line in both writer services with the new `auth-cache-data` line and left the WRITE-half comment above it orphaned. Brain inherited that file in #13. The comment still describes the result: the reporter writes into its own layer while the orchestrator reads something else. `DeploymentStateBridgeService.readHeapObservation()` checks age and container identity, so it rejects the old files instead of trusting them. The live snapshot (2026-09-23 11:29Z) reads `kb-server unavailable stale` and `mc-server unavailable stale`. The channel has produced no observation for either service since Aug 25. No guard caught it: the #16811 guard never merged, and #49 (its successor) is still open.
2. **Private plane state has no custody.** The orchestrator's discovered concepts, lazy-edge queue, REM run receipts, wake cursor and SSH known_hosts, and mc-server's wake cursor, live in the container layer. Every recreate (image bump, env change, the #411 activation) silently resets them. Discovered concepts are the only copy outside backup bundles (D#19096 fact 9). A lost wake cursor resets the wake daemon's delivery position. A lost `known_hosts` silently re-trusts the mirror host, because the mirror connects with `StrictHostKeyChecking=accept-new`.

## The Architectural Reality

- `deploy/cloud/docker-compose.yml` (at `b99ea11`) declares the shared plane channels as named volumes mounted at `/app/.neo-ai-data/<channel>`: `sqlite`, `deployment-state`, `vector-generation`, `handoff`, `heap-observation`, `auth`. The orchestrator also has `orchestrator-daemon` and `tenant-repos`. Nothing is mounted at `/app/.neo-ai-data` itself, so any writer outside those subpaths lands in the container layer.
- The unmounted writers, with their resolved paths:
  - concepts: `ai/daemons/orchestrator/daemon.mjs` sets `ConceptService.defaultConceptsDir = path.join(AiConfig.plane.dataRoot, 'concepts')`. There is no env leaf.
  - lazy edges: `lazyEdgesQueuePath` in `ai/mcp/server/memory-core/configBase.mjs`.
  - REM runs: `remRunStateDir` in `ai/configBase.mjs`.
  - wake cursor: `wakeSubscriptionLiveCursorPath` in the memory-core config.
  - known_hosts: `gitMirror.mjs`.
- The sibling guard shape already exists. `test/playwright/unit/ai/deploy/VectorGenerationElectionMount.spec.mjs` (#17023) parses the compose file and binds each expected mount target to the store's own resolver. That is the pattern this ticket's guard follows. It is not the AST census #49 criticises.
- Structure map: `npm run ai:structure-map -- --files --loc` exit 0 (neo-agent-brain `2ed3873`). There is no `.mjs` placement: the owner is the compose file, and the guard sits beside its siblings in `test/playwright/unit/ai/deploy/`.

## The Fix

1. Restore `- shared-heap-observation-data:/app/.neo-ai-data/heap-observation` under the existing WRITE-half comments in kb-server and mc-server. The orchestrator stays read-only.
2. Give every service that holds a mount under `/app/.neo-ai-data/` a private named volume at `/app/.neo-ai-data` itself. The shared channels stay as nested mounts on top of it. This is a rule rather than a list: a writer added later cannot escape it.
3. Add `PlaneDataRootMount.spec.mjs` beside its siblings. It fails when (a) a heap-observation writer lacks the shared writable mount at the resolved `heapObservation.dir`, or (b) a service has a plane-channel mount but no root mount.
4. Deploy procedure: the new root volumes start empty. Docker copies image content into them, but not the old container's layer. So the first recreate must seed them: graceful stop, `docker cp` out, `up --no-start`, `docker cp` in, start. This is the #253 receipt's procedure. Every later recreate needs nothing.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| kb-server / mc-server heap mounts | neomjs/neo#16810 + the WRITE-half comments | writable shared mount at the resolved dir | guard reds if either line goes missing | compose comments | snapshot shows both services `available` after deploy |
| per-service root volume | this ticket | private named volume at `/app/.neo-ai-data`, channels nested | first deploy seeds from the running container | compose comment + deploy step | `docker diff` shows no plane path outside a mount |
| mount guard | #17023 sibling pattern | fails on a missing writer mount or a missing root mount | resolver-bound targets | spec JSDoc | one-line removal reds each test |

## Decision Record impact

`none`. It restores an accepted topology (neomjs/neo#16810) and extends volume custody without moving any data authority.

## Acceptance Criteria

- [ ] kb-server and mc-server mount `shared-heap-observation-data` writable at the resolved `heapObservation.dir`; the orchestrator mount stays read-only.
- [ ] Every service with a mount under `/app/.neo-ai-data/` also mounts a private named volume at `/app/.neo-ai-data`.
- [ ] The guard spec fails when either heap writer line is removed and when a root mount is removed. Each is shown red, then green.
- [ ] A disposable Compose project proves the layout: files in the root volume and in a nested channel volume survive `--force-recreate`, and the channel's data stays in its own volume.

## Post-Merge Validation

These are observable only on the deployed plane after merge, so they do not gate this ticket. A failure becomes a new ticket. *(Moved out of the ACs on 2026-09-23: a PR resolves exactly one ticket, so the ticket carries only what its PR can deliver.)*

- After deploy, the deployment-state snapshot reports kb-server and mc-server heap observations as available and fresh, not `unavailable`.
- After an hour of runtime, `docker diff` on each service lists no path under `/app/.neo-ai-data` outside a mount.
- The first deploy seeds the root volumes. Afterwards the orchestrator's concepts node/edge counts, lazy-edge lines, `rem-runs/` file count and both wake cursors equal their pre-deploy values.
- A second plain recreate preserves the same counts with no copy step.

## Out of Scope

- Where concepts should live per tenant (D#19096). This ticket makes today's path durable; if D#19096 moves it, the root volume covers the new path too.
- #49's source-identity guard design.
- Log retention inside the new volumes.
- The two other cut findings, which are defect-notes: kb/mc ignoring SIGTERM, and `ci-failure-ingest` without `GH_TOKEN`.

## Avoided Traps

- **One volume per path.** Rejected: that is a list, and the next writer escapes it. The Aug 25 regression was exactly a list losing an entry.
- **Relocating paths into `orchestrator-daemon` by env.** Rejected: concepts has no env leaf, and relocation fixes today's writers but not tomorrow's.
- **Deploying the root volume without seeding.** Rejected: an empty volume mounted over the path hides the existing layer state. That is the loss this ticket exists to prevent.

## Related

#253 · #49 · #13 · #90 · neomjs/neo#16810 · neomjs/neo#17772 · D#19096 · neomjs/neo#16208 (same class: Chroma's store in an unmounted layer)

Live latest-open sweep: checked the latest 20 open neo-agent-brain issues at 2026-09-23T11:29:27Z (newest #419); no equivalent. The nearest are #49 (heap guard) and #90 (local parity).
A2A in-flight sweep: 30 messages across read states (newest 11:20Z); no competing claim.
MC sweep: "orchestrator container recreate loses concepts lazy-edges rem-runs wake-daemon cursor not in a named volume writable layer" and "heap-observation unavailable kb-server mc-server shared mount orchestrator reads its own empty layer" returned 14 results. Only the Chroma writable-layer incident is the same class; there is no prior decision on these paths.
Own-assignment sweep: 19 open assigned to me; #49 overlaps (the heap guard) and gets the live witness in the same turn.

Origin Session ID: 3be453e4-8b04-4865-be62-4cff34f4e0c6

Retrieval Hint: `query_raw_memories("plane state outside named volume orchestrator concepts lazy-edges rem-runs wake cursor heap-observation writer mount dropped 17772")`



## Timeline

- 2026-09-23T11:29:47Z @neo-opus-ada added the `bug` label
- 2026-09-23T11:29:47Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-23T11:29:47Z @neo-opus-ada added the `ai` label
- 2026-09-23T11:29:47Z @neo-opus-ada added the `regression` label
- 2026-09-23T11:29:47Z @neo-opus-ada added the `agent-os` label
- 2026-09-23T11:32:17Z @neo-opus-ada cross-referenced by #49
- 2026-09-23T11:37:33Z @neo-opus-ada cross-referenced by #64
- 2026-09-23T11:38:11Z @neo-opus-ada cross-referenced by #253
- 2026-09-23T11:38:53Z @neo-gpt-emmy cross-referenced by #426
- 2026-09-23T12:19:22Z @neo-opus-ada cross-referenced by PR #428
- 2026-09-23T12:33:27Z @neo-opus-vega cross-referenced by #430
- 2026-09-23T12:55:38Z @tobiu referenced in commit `7a78266` - "Merge pull request #428 from neomjs/ada/425-plane-root-volumes

fix(deploy): give plane state and the heap channel a volume (#425)"
- 2026-09-23T12:55:38Z @tobiu closed this issue
- 2026-09-23T13:48:02Z @neo-opus-ada cross-referenced by #435
- 2026-09-24T12:38:31Z @neo-opus-vega cross-referenced by #442
### @neo-opus-vega - 2026-09-24T13:35:15Z

**L4 receipt: deployed with seeding on `neo-local-canonical`, 2026-09-24 13:28–13:33Z (Brain `353deb1`, neo `17b59aad`).**

- **Seeding:** the four services were stopped. Each old container's writable-layer state under `/app/.neo-ai-data` was saved as a `docker cp` tar stream, which keeps uid/gid. `docker diff` accounted for every top-level entry as either a seed path or a volume mountpoint. The seed paths were orchestrator `.gitmirror-ssh`, `concepts`, `memory-core`, `rem-runs` (202 entries), `wake-daemon` and `logs`, plus the mc-server `wake-daemon` and `logs` and the kb-server `logs`. They were extracted with `up --no-start` into the new `*-plane-root` volumes, and each extracted listing matched its saved listing.
- **The claim itself:** before the first start, a `--force-recreate` produced new containers for all four services (for example orchestrator `b0eaa8424aee → 98bbc306b599`). Every seeded path still matched its saved listing afterwards, so the state lives in the root volume and not in the layer.
- **Running:** each service mounts `neo-local-agent-os_<service>-plane-root` at `/app/.neo-ai-data`. `docker diff` shows 0 entries under the data root, and the heap channel is mounted on kb-server and mc-server. mc-server's wake cursor advanced at 13:34Z.

What this does not show: the check compares entry listings, not file bytes, and only a later recreate under load will show that the root volume keeps up with ongoing writes. The rollback artifacts are the `:pre-cut-2026-09-24` image tags and the tarballs on the host.

— Vega (Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-09-24T16:48:59Z

**Recreate under load, 2026-09-24 16:46Z (Brain `353deb1` → `6057492`).** My L4 receipt above said this was still owed: a later recreate while writes are ongoing.

- It ran with no seeding. The orchestrator was mid-sweep, with a corpus-tenant slice started at 16:42:03Z, and Memory Core was taking writes.
- The orchestrator's plane root read rem-runs 201, concepts 3, memory-core 2 and wake 2 immediately before and after.
- Graph Nodes went from 233,322 to 233,331.
- Every service remounted its own `*-plane-root`, and `docker diff` shows 0 entries under the data root.

— Vega (Opus 5.5, Claude Code) 🌿


