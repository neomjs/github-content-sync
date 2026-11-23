---
id: 1435
title: kb-server has no write gate for the KB defrag window
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-02T17:38:13Z'
updatedAt: '2026-11-23T17:47:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1435'
author: neo-opus-vega
commentsCount: 2
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
  - '[ ] 419 Retire replaced legacy core rows by profile receipt'
  - '[ ] 1432 No defrag heal operation runs the KB''s store-bloat diagnosis'
closedAt: '2026-11-23T17:47:25Z'
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

**Second consumer (added 2026-11-06):** #1501's exact-row retirement operation, which #419 runs on the deployed KB, needs the same fence. It acquires the lease declaring KB-write exclusion and starts backup, plan and apply only once kb-server's health reports the gate closed for its own lease id with zero in-flight writes. It holds the lease through apply. The gate and acknowledgement above serve both consumers unchanged, so nothing here is specific to #1432.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| kb-server KB writes: the 6 incremental sites, plus `deleteCollection` and the shadow swap's create and promote (9 writes through `kbWriteGate.runWrite`; `assertOpen` pre-checks before embedding and before a fresh shadow build) | #1432 convergence item 2 | refuse with `KB_MAINTENANCE_IN_PROGRESS` while a KB-exclusive lease is held | writes proceed when no such lease is held; `DatabaseService.importDatabase` / `truncateDatabase` stay ungated as the lease holder's own restore path (#1432) | JSDoc on the gate | specs per write class (upsert, delete) |
| heavy-maintenance lease record | `HeavyMaintenanceLeaseService.mjs` | the holder can declare KB-write exclusion (`kbWriteExclusive: true`); the record carries a public `leaseId` beside the secret release `token`; a reader in another container inspects it with `crossNamespace: true` (deadline liveness only) | absent ⇒ no exclusion | JSDoc | spec |
| kb-server health payload | `knowledge-base/HealthService.mjs` | adds `{gate, leaseId, inFlightWrites}` | gate `open`, count 0 when no lease | the health schema | spec + live read after the cut |
| `ingest_source_files`, `manage_knowledge_base` | `knowledge-base/openapi.yaml` | return the refusal as a retryable error naming the window | none | tool handbook | spec |
| kb-server compose mounts | `deploy/cloud/docker-compose.yml` | the lease directory, `:ro` | none | compose comment | `docker inspect` after the cut |

*Rows 1 and 2 synced to PR #1909's shipped contract at `a609a46ad3` (Ada, review, 2026-11-23).*

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
- 2026-11-02T17:38:30Z @neo-opus-vega added parent issue #1416
- 2026-11-02T17:38:36Z @neo-opus-vega marked this issue as blocking #1432
- 2026-11-02T17:39:50Z @neo-opus-vega cross-referenced by #1432
- 2026-11-02T17:48:26Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T18:14:15Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
- 2026-11-02T19:41:48Z @neo-gemini-maryam cross-referenced by #306
- 2026-11-05T11:16:41Z @neo-opus-vega cross-referenced by #66
- 2026-11-05T11:17:31Z @neo-opus-vega cross-referenced by #55
- 2026-11-05T11:18:21Z @neo-opus-vega cross-referenced by #60
- 2026-11-05T11:35:02Z @neo-opus-vega cross-referenced by PR #1478
- 2026-11-06T04:00:50Z @neo-gpt-euclid cross-referenced by PR #1502
- 2026-11-06T04:02:27Z @neo-gpt-euclid cross-referenced by #419
- 2026-11-06T04:02:32Z @neo-gpt-euclid marked this issue as blocking #419
- 2026-11-06T12:57:41Z @neo-gpt-euclid cross-referenced by #1507
- 2026-11-23T06:33:39Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-23T06:33:40Z @neo-gemini-maryam unassigned from @neo-opus-vega
### @neo-gemini-maryam - 2026-11-23T06:33:41Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-vega`
**New assignees:** `@me`
**Reason:** Accepted @neo-opus-vega handoff offer MESSAGE:95b7f3ac-0102-483c-b09a-8973344d5ac0

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

### @neo-gemini-maryam - 2026-11-23T07:06:58Z

## Ticket Intake & Structural Pre-Flight (`dev@84e2b1fb`, stacked on `#1908` / `96667e41`)

Advancing to `#1435` (second leaf of the `#1434` → `#1435` → `#1432` KB defrag chain claimed from `@neo-opus-vega` in `MESSAGE:95b7f3ac-0102-483c-b09a-8973344d5ac0`).

### §0 Intent & Prescription Check (Accept + Sharpen)
- **Premise verified against current Brain `dev@84e2b1fb`:**
  - All 6 Chroma KB write sites in `kb-server` are live and unguarded against defrag windows:
    1. `ai/services/knowledge-base/IngestionService.mjs:587` (`collection.delete({ids: Array.from(ids)})`)
    2. `ai/services/knowledge-base/VectorService.mjs:1040` (`collection.upsert`)
    3. `ai/services/knowledge-base/VectorService.mjs:1492` (`collection.upsert`)
    4. `ai/services/knowledge-base/VectorService.mjs:1583` (`collection.upsert`)
    5. `ai/services/knowledge-base/VectorService.mjs:2712` (`collection.delete({ids: idsToDelete})`)
    6. `ai/services/knowledge-base/VectorService.mjs:2751` (`collection.delete({ids: replacedIds})`)
  - `deploy/cloud/docker-compose.yml` mounts `orchestrator-state:/app/.neo-ai-data/orchestrator-state` only into `orchestrator` (`AiConfig.orchestrator.dataDir`); `kb-server` cannot observe `heavy-maintenance-lease.json`.
  - Parent `#1416` (`enhancement`, `ai`, `performance`, `agent-os` — non-epic investigation issue) is active and converged on the 4-leaf order `#1433` (closed) → `#1434` (`PR #1908` open) → `#1435` → `#1432`.
- **Two substrate traps surfaced during intake V-B-A (sharpening implementation contract):**
  1. **Cross-container `bootId` + `pid` namespace trap (`heavyMaintenanceLeasePrimitives.mjs:136, 164`):** `orchestrator` stamps `lease.bootId = os.hostname()` (orchestrator container hostname) and `lease.pid = process.pid` (orchestrator PID namespace). Inside `kb-server`'s separate container, `os.hostname()` differs (`!== lease.bootId`) and `isPidAlive(lease.pid)` probes `kb-server`'s own PID namespace (`ESRCH`). Calling `inspectHeavyMaintenanceLease` with default same-container discriminators from `kb-server` would classify every live orchestrator lease as `stale: true, active: false`! The `:ro` cross-container reader must evaluate deadline-based liveness (`expiresAt` / `acquiredAt + staleAfterMs`, with `crossNamespace: true` / `currentBootId: null, isPidAlive: null`) while same-container `orchestrator` callers retain `bootId` + `pid` discrimination.
  2. **5-minute `#cachedHealth` bypass trap (`ai/services/knowledge-base/HealthService.mjs:744-756`):** `HealthService#healthcheck()` serves `this.#cachedHealth` for up to 5 minutes when `healthy`. Computing `{gate, leaseId, inFlightWrites}` only inside `#performHealthCheck()` would return a stale `gate: 'open'` or stale `inFlightWrites` to `#1432`'s drain check and `#1501`'s retirement fence. The write-gate block must project live on every `healthcheck()` call (alongside `runtimeFreshness` and `#applyEmbeddingProbe`), never from `#cachedHealth`.
- **Prescription checked:** `ai/services/knowledge-base/helpers/kbWriteGate.mjs` + `ai/daemons/orchestrator/services/heavyMaintenanceLeasePrimitives.mjs` + `ai/services/knowledge-base/{IngestionService,VectorService,HealthService}.mjs` + `ai/mcp/server/knowledge-base/openapi.yaml` + `deploy/cloud/docker-compose.yml` + `learn/agentos/decisions/0026-recovery-actuator.md` — owns the concern.
- **Core-Idiom Pre-Flight (§1.9.6):** Read `src/core/Base.mjs`, `src/Neo.mjs`, and `src/state/Provider.mjs`, plus `ADR 0019` (`0019-aiconfig-reactive-provider-ssot.md`). `kb-server` resolves `resolveHeavyMaintenanceLeasePath({dataDir: aiConfig.orchestrator.dataDir})` at the use site with zero new `AiConfig` leaves (`B3`/`B5`/`C1` clean).

### §1.8 Ticket Reality & ADR Audit
- `Ticket age`: `createdAt: 2026-11-02T17:38:13Z`, `updatedAt: 2026-11-23T06:33:41Z`.
- `Bot stale-band`: `pre-stale` (`stale: false`, `no auto close: false`).
- `Currency / successor-risk evidence`: Checked `git log --since=2026-11-02T17:36:00Z` across `ai/services/knowledge-base/`, `ai/mcp/server/knowledge-base/`, `HeavyMaintenanceLeaseService.mjs`, and `deploy/cloud/docker-compose.yml` (`#1494` `ab82c40f` added `createLeaseYieldVoter`; `#1598` `443cc90d`, `#1599` `c415930b`, `#1744` `6cfdf63e` shifted line offsets slightly in `IngestionService.mjs` / `VectorService.mjs` while keeping the exact 6 Chroma write sites intact). Live `ask_knowledge_base` (`type='ticket'`) and open PR queue (`#1816`, `#1908`) show zero duplicate write-gate work.
- `ADR successor-risk`: `adr-amendment-required` — artifact `#1435/2026-11-02`; ADR `neomjs/neo-agent-brain#adr:recovery-actuator` (`Accepted/2026-06-23`, amended in `#1908` for `#1434` §2.10); evidence `learn/agentos/decisions/0026-recovery-actuator.md` §2.3/§2.10 + `#1432` convergence `6582932837`; route `amendment-required` (declares `kb-server`'s `:ro` lease-observation mount alongside `#1434`'s §2.10; stacked on `agent/1434-kb-dead-hnsw-elements` until `#1908` merges).
- `Verdict`: `valid-as-written` (positive ROI — prevents silent write/delete loss during `#1432` KB compaction and `#1501` exact-row retirement without taking KB reads offline).

### Structural Pre-Flight (§1 Fast-Path)
> *"Pre-Flight (structural fast-path): authoring `ai/services/knowledge-base/helpers/kbWriteGate.mjs` matches sibling pattern of `ai/services/memory-core/helpers/quarantineStore.mjs` and existing helpers in `ai/services/knowledge-base/helpers/` (`GitMirror.mjs`, `coreCorpusProfileRunner.mjs`); sibling-file-lift applies; no novel directory choice (`test/playwright/unit/ai/services/knowledge-base/helpers/kbWriteGate.spec.mjs` mirrors under `test/playwright/unit/ai/`)."*

🧭 **Maryam** · `@neo-gemini-maryam` · Gemini 4 Carbon · Antigravity

- 2026-11-23T08:58:02Z @neo-gemini-maryam cross-referenced by PR #1909
- 2026-11-23T09:47:25Z @neo-gemini-maryam referenced in commit `a609a46` - "test(knowledge-base): align OpenAPI union and Compose mount specs with KB write gate (#1435)"
- 2026-11-23T12:30:01Z @neo-gemini-maryam referenced in commit `7f9bbb1` - "feat(knowledge-base): gate kb-server writes during KB maintenance window (#1435)"
- 2026-11-23T12:30:01Z @neo-gemini-maryam referenced in commit `177dd9a` - "test(knowledge-base): align OpenAPI union and Compose mount specs with KB write gate (#1435)"
- 2026-11-23T17:47:24Z @tobiu referenced in commit `5073c82` - "feat(knowledge-base): gate kb-server writes during KB maintenance window (#1435) (#1909)

* feat(knowledge-base): gate kb-server writes during KB maintenance window (#1435)

* test(knowledge-base): align OpenAPI union and Compose mount specs with KB write gate (#1435)"
- 2026-11-23T17:47:26Z @tobiu closed this issue

