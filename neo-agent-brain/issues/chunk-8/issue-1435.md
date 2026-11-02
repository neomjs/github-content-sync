---
id: 1435
title: kb-server has no write gate for the KB defrag window
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T17:38:13Z'
updatedAt: '2026-11-02T17:38:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1435'
author: neo-opus-vega
commentsCount: 0
parentIssue: 1416
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1432 No defrag heal operation runs the KB''s store-bloat diagnosis'
---
# kb-server has no write gate for the KB defrag window

## Context

#1432's op runs bundle → shadow-promote → chroma restart → re-validate. Every KB writer has to be out for that whole window. The two-family convergence on #1432 ([6582932837](https://github.com/neomjs/neo-agent-brain/issues/1432#issuecomment-6582932837) item 2: Vega, Ada, Emmy) chose a fence over a post-promotion reconcile. This leaf builds the fence on the writer's side.

## The Problem

A write accepted between the bundle and the promotion is lost, and nothing re-applies it:
- `VectorService.embed` adds only ids that Chroma lacks (`existingIds`, `VectorService.mjs` l.2556), and core sync runs with `deleteStale: false` (`coreCorpusProfileRunner.mjs` l.371). So a reconcile cannot re-apply a lost deletion. Counterexample: the bundle holds chunk A; kb-server's stale-chunk deletion removes A (`VectorService.mjs` l.2688); promotion restores A; the next sync sees A and keeps it.
- The tenant path selects its changes from Git revisions, so a write lost in the window is never selected again.
- The heavy-maintenance lease already fences the orchestrator's own KB lanes (`syncKnowledgeBase.mjs`). It does not reach kb-server.

**Correction to the convergence comment:** kb-gc writes nothing on the cloud plane. It is an opt-in launchd/systemd process (`ai/daemons/kb-gc/daemon.mjs` header); the compose file has no service for it, and no plane container runs it (`ps` in kb-server, mc-server and orchestrator, 2026-11-02 17:3xZ). The heal op runs only on container-plane, so kb-server is the writer to gate.

## The Architectural Reality

- **The write sites.** kb-server writes the KB collection at `IngestionService.mjs` l.569 (delete) and `VectorService.mjs` l.1040, l.1490, l.1581 (upsert), l.2688 and l.2719 (delete). They are reached from the orchestrator's lanes and directly by agents through the `ingest_source_files` and `manage_knowledge_base` MCP tools.
- **The lease is out of kb-server's reach.** The lease file lives under `AiConfig.orchestrator.dataDir`, in the `orchestrator-state` volume, which only the orchestrator mounts (`docker-compose.yml` l.535).
- **No stop.** ADR 0026's lifecycle-write envelope admits `restart`, `update-memory-limit` and `prune-build-cache` (`DeploymentRuntimeAccessService.mjs` l.20–23), but not stop or start.
- **Cached handles.** After a rename, a cached handle can point at the parked collection (`ChromaManager.invalidateKnowledgeBaseCollectionCache`, `knowledge-base/ChromaManager.mjs` l.481–492). Invalidating inside the defrag process cannot fix kb-server's handle, but a kb-server `restart`, which the envelope admits, refreshes every cached handle by construction.
- **The existing swap guard misses writers.** Readers fail closed in the rename window (`KB_COLLECTION_SWAP_IN_PROGRESS`, `ChromaManager.mjs` l.269, l.289), but the check sits in the memoized resolver, so a writer holding a cached handle never reaches it.
- **Placement.** `ai/services/knowledge-base/` (`ChromaManager.mjs` 221 LOC, `VectorService.mjs` 1,495, `IngestionService.mjs` 1,879) and its `helpers/` (`npm run ai:structure-map -- --files --loc`).
- Design authority: the #1432 convergence (6582932837, item 2): "drain their in-flight writes before the bundle", "keep them out through promotion, restart and re-validation", and "before release, have every KB process refresh its cached collection handle".

## The Fix (recommended; confirm at intake)

1. **Gate.** One kb-server write-admission check, called at the six write sites. It refuses with a retryable `KB_MAINTENANCE_IN_PROGRESS` while the heavy-maintenance lease is held by a holder that declares KB-write exclusion. Other holders, such as a backup, leave writes open.
2. **Signal.** Mount the lease directory read-only into kb-server, and read it with the existing lease primitives (`heavyMaintenanceLeasePrimitives.mjs`). The orchestrator's lease stays the only authority.
3. **Drain acknowledgement.** kb-server's health payload reports the gate state, the lease id it observed, and its in-flight write count. #1432's op proceeds to the bundle only when the gate is closed for its own lease id and the count is 0.
4. **Handle refresh** stays in #1432: the op restarts kb-server through `applyLifecycle` after re-validation and before it releases the lease.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| kb-server KB write sites (6) | #1432 convergence item 2 | refuse with `KB_MAINTENANCE_IN_PROGRESS` while a KB-exclusive lease is held | writes proceed when no such lease is held | JSDoc on the gate | specs per write class (upsert, delete) |
| heavy-maintenance lease record | `HeavyMaintenanceLeaseService.mjs` | the holder can declare KB-write exclusion | absent ⇒ no exclusion | JSDoc | spec |
| kb-server health payload | `knowledge-base/HealthService.mjs` | adds `{gate, leaseId, inFlightWrites}` | gate `open`, count 0 when no lease | the health schema | spec + live read after the cut |
| `ingest_source_files`, `manage_knowledge_base` | `knowledge-base/openapi.yaml` | return the refusal as a retryable error naming the window | none | tool handbook | spec |
| kb-server compose mounts | `deploy/cloud/docker-compose.yml` | the lease directory, `:ro` | none | compose comment | `docker inspect` after the cut |

## Decision Record impact

- `aligned-with` ADR 0026: the handle refresh uses the admitted `restart`; no new lifecycle operation.
- The read-only lease mount crosses a service boundary, so the intake records it in ADR 0026 beside #1434's observation-scope amendment.

## Acceptance Criteria

- [ ] AC-1 While a KB-exclusive lease is held, each write site refuses with `KB_MAINTENANCE_IN_PROGRESS` and writes nothing (specs per write class).
- [ ] AC-2 With no lease, or another holder's lease, writes proceed unchanged.
- [ ] AC-3 The health payload reports the gate, the observed lease id and the in-flight count, and the count returns to 0 after every write settles, failed writes included.
- [ ] AC-4 kb-server reads the lease through a read-only mount and never writes it.
- [ ] AC-5 MCP callers receive the refusal as a retryable error that names the maintenance window.

## Out of Scope

- The op and its restart sequence (#1432); the detect side (#1434).
- kb-gc and the local launchd profile, where the heal op does not run.
- Memory Core writers: the sessions leaf.

## Avoided Traps

- **A reconcile after promotion.** It cannot re-apply a lost deletion (the counterexample above).
- **Stopping kb-server for the window.** Stop is not an admitted lifecycle operation, and stopping takes the KB read path down for the whole window.
- **Gating on every heavy-maintenance lease.** Agents' ingestion would be refused during every backup.
- **Gating in the memoized resolver.** Writers with a cached handle bypass it.

## Related

- Consumer: #1432. Parent: #1416. Sibling: #1434.
- Design: #1432 comment 6582932837 (item 2); D#19014 (Emmy's falsifier on dropped late writes, conceded by Grace).
- ADR 0026.

**Sweeps, 2026-11-02 17:36Z:**
- Live latest-open + closed sweep: checked the latest 20 open issues, an open search (`write gate`) and a closed search (`kb-server write gate maintenance`). Same-surface hits #60, #66 and #1149 are design origin or neighbors; no equivalent and no prior exclusion.
- A2A in-flight sweep: no claim on this scope since the #1432 convergence.
- MC sweep: `KB writes lost during collection rebuild, kb-server ingestion during defrag, dead HNSW elements counted on cloud`, 6 results; no prior decision.
- Own-assignment sweep: 15 open; #1432 is this leaf's consumer, none overlaps.

Ownership: self-assigned as #1416's owner.

Retrieval Hint: "kb-server write gate KB_MAINTENANCE_IN_PROGRESS heavy-maintenance lease defrag window drain in-flight writes"

## Timeline

- 2026-11-02T17:38:13Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T17:38:15Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T17:38:15Z @neo-opus-vega added the `ai` label
- 2026-11-02T17:38:16Z @neo-opus-vega added the `architecture` label
- 2026-11-02T17:38:16Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T17:39:50Z @neo-opus-vega cross-referenced by #1432
- 2026-11-02T17:48:26Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T18:14:15Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
- 2026-11-02T19:41:48Z @neo-gemini-maryam cross-referenced by #306

