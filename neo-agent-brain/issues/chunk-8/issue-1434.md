---
id: 1434
title: No producer measures the KB's dead HNSW elements on the cloud plane
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - performance
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-02T17:37:29Z'
updatedAt: '2026-11-23T12:18:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1434'
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
blockedBy:
  - '[x] 1433 The data-integrity sweep never completes: its lane has no state envelope'
blocking:
  - '[ ] 1432 No defrag heal operation runs the KB''s store-bloat diagnosis'
closedAt: '2026-11-23T12:18:48Z'
---
# No producer measures the KB's dead HNSW elements on the cloud plane

## Context

#1432 (the KB defrag heal op, #1416 AC-3) can only run when a diagnosis dispatches it. Its intake ([6582630907](https://github.com/neomjs/neo-agent-brain/issues/1432#issuecomment-6582630907)) found that no such diagnosis exists. A two-family convergence ([6582932837](https://github.com/neomjs/neo-agent-brain/issues/1432#issuecomment-6582932837): Vega, Ada, Emmy) fixed the instrument's shape, and this leaf builds it. Measured 2026-11-02: `neo-knowledge-base` holds 253,743 HNSW elements for 86,560 live records ([#60, 6569290980](https://github.com/neomjs/neo-agent-brain/issues/60#issuecomment-6569290980)).

## The Problem

Nothing on the cloud plane can produce a KB dead-element diagnosis:
- **No producer.** `buildStoreBloatDiagnosis` has no production caller, and it measures something else: store size for the Memory Core service (`storeBloatDiagnosis.mjs` l.36: `storeSizeBytes`, `serviceId`), not dead elements per collection.
- **Wrong routing.** The assembler folds every store-level signal into one row keyed by the service id, `mc-server` (`dataIntegrityEvidenceAssembler.mjs` l.79–91). A KB-only op would receive `collection: 'mc-server'` and refuse it.
- **No store access.** `chroma-data` mounts only into `chroma` (`deploy/cloud/docker-compose.yml` l.36). `engines.chroma.dataDir` resolves to a path that does not exist in the orchestrator container, and the image ships no `sqlite3` binary.
- **No admitted scope.** The runtime access read envelope admits only `inspect`, `logs`, `stats`, `events` and `system-df` (`DeploymentRuntimeAccessService.mjs` l.12–18), and ADR 0026 §2.3 confines the Docker socket to known-target envelopes with no exec. A read-only mount is therefore a new observation scope, and ADR 0026 has to declare it before any code.
- **An all-or-nothing gatherer.** `dataIntegrityEvidenceGatherer` awaits the coverage gatherer first (`Orchestrator.mjs` l.1406), which shells out to `sqlite3`. Today every gather throws there, `gatherAndDiagnose` returns `probe-unavailable`, and a new diagnosis beside coverage would never reach the classifier.
- **Prerequisite:** the sweep that runs the gatherer has never completed a run (#1433).

## The Architectural Reality

- **What to read.** `chroma.sqlite3` is 4.47 GB on a rollback journal, not WAL: header bytes 18–19 read `1 1`, and there are no `-wal` or `-shm` files ([6582788745](https://github.com/neomjs/neo-agent-brain/issues/1432#issuecomment-6582788745)). A copy per sweep cannot fit the guest disk. The measure needs only the segment registry (two small tables) and each KB vector segment's 100-byte `header.bin`.
- **The reader contract** (Emmy's conditions, 6582932837 item 1):
  - bind the observed endpoint, the compose service and container incarnation, the physical volume and the KB segment identity;
  - open the registry read-only (`better-sqlite3` with `readonly`, plus `PRAGMA query_only`). A hot rollback journal that a read-only open cannot replay ends as `unverified`, which skips the sweep;
  - interpret `header.bin` for Chroma 1.5.9 only, and report any other version as `unknown`;
  - keep HNSW occupancy and the live `count()` distinct. **Occupancy above the live count IS the signal.** `unknown` is reserved for incoherent observations: an identity, version or checkpoint mismatch.
- **Config (ADR 0019).** The reader reads `AiConfig.engines.chroma.dataDir` at the use site, as the coverage gatherer already does (§5.1). It adds no leaf and threads no value into another consumer (B5). The threshold is the existing `gcDefragThreshold` leaf (`configBase.mjs` l.2802, 0.10), read at the use site; no new threshold.
- **Routing.** A per-collection dead-element fact → the assembler's row for the KB collection (not the `mc-server` store row) → `classifyDataIntegrityMode`, which already maps `store-bloat` to `defrag` (`dataIntegrityModeClassifier.mjs` l.49, l.124) → `dispatchHeal`. No new dispatcher.
- **Placement.** The diagnosis builders live in `ai/daemons/orchestrator/services/` (`dataIntegrityCoverageDiagnosis.mjs` 32 LOC, `storeBloatDiagnosis.mjs` 47 LOC; `npm run ai:structure-map -- --files --loc`). The reader's file is placed at intake through `structural-pre-flight`.
- Design authority: ADR 0027 admits `store-bloat → defrag` as "maintenance, not corruption recovery". ADR 0026 §2.3 declares the observation envelopes that this leaf extends.

## The Fix

1. **ADR 0026 amendment.** Declare a read-only store-observation scope: the `chroma-data` volume mounted `:ro` into the orchestrator and read only by the dead-element reader under the contract above. No write, no exec, no other consumer.
2. **Compose.** The `:ro` mount in `deploy/cloud/docker-compose.yml`, at the path `AiConfig.engines.chroma.dataDir` resolves to.
3. **Producer.** A pure dead-element diagnosis builder beside `storeBloatDiagnosis.mjs`. It returns `null` below the threshold, as its sibling does when a store is not bloated. The reader is its injected gatherer.
4. **Gatherer isolation (decide at intake).** Recommended: run the dead-element gatherer independently of coverage, so a failed coverage probe yields `probe-unavailable` for coverage alone while the dead-element diagnosis still reaches the classifier. This keeps `gatherAndDiagnose`'s rule that nothing is actioned on a failed probe's evidence. *Falsifier:* any classifier mode that combines coverage and store evidence for one collection, which a partial gather could mis-key. The alternative, moving coverage onto the same read-only reader, would switch on coverage-driven heals over live Memory Core collections for the first time. That needs its own observed rollout, so it is out of scope here.
5. **Assembler.** Route the dead-element fact to the KB collection's own row.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| ADR 0026 observation scopes | §2.3 (B1 envelopes) | adds one read-only store-observation scope | none: the reader refuses to run without it | the ADR amendment | diff |
| orchestrator compose mounts | `deploy/cloud/docker-compose.yml` | `chroma-data` mounted `:ro` | none | compose comment | `docker inspect` mounts after the cut |
| dead-element diagnosis (new) | ADR 0027 `store-bloat → defrag` | a KB per-collection fact above `gcDefragThreshold` | `null` below it; `unverified` on a hot journal; `unknown` on incoherent identity | JSDoc | unit specs on fixture registry rows and `header.bin` bytes |
| `assembleDataIntegrityEvidence` | `dataIntegrityEvidenceAssembler.mjs` l.79–91 | routes the fact to the KB row | the store-level row stays for the MC store signals | JSDoc | spec: classifier output `defrag` for `neo-knowledge-base` |

## Decision Record impact

- `amends` ADR 0026: a new read-only observation scope beside the §2.3 envelopes.
- `aligned-with` ADR 0027 (`store-bloat → defrag`) and ADR 0019 (use-site reads, no new leaf).

## Acceptance Criteria

- [ ] AC-1 ADR 0026 declares the read-only store-observation scope: who mounts it, who reads it, and what it may never do.
- [ ] AC-2 The orchestrator mounts `chroma-data` `:ro` at the path `AiConfig.engines.chroma.dataDir` resolves to; no config leaf is added.
- [ ] AC-3 On fixture registry rows and `header.bin` bytes, the producer emits a KB dead-element fact when the dead fraction, (occupancy − live count) / occupancy, exceeds `gcDefragThreshold`. It returns `null` below it, `unverified` on a hot journal, and `unknown` on a version or identity mismatch. No code path copies the full database.
- [ ] AC-4 The assembler routes the fact to the KB's own row, and the classifier returns `defrag` for `neo-knowledge-base` (spec).
- [ ] AC-5 A coverage probe failure no longer prevents the dead-element diagnosis from reaching the classifier, and no coverage-class heal runs on the failed probe (spec).
- [ ] AC-6 *(post-merge, live)* After the cut, one sweep on this plane records the KB's occupancy and live count, and its `defrag` dispatch records `deferred: no heal operation wired` until #1432 lands. Posted on this ticket.

## Out of Scope

- The heal op itself (#1432) and the KB writer gate (its sibling leaf).
- Turning on coverage-driven heals over live Memory Core collections.
- `neo-agent-sessions` (about 5.2 GiB dead): the sessions leaf, behind Ada's three Memory Core prerequisites on #60.

## Avoided Traps

- **A full copy per sweep.** 4.47 GB on a guest disk that had 5.6 GiB free.
- **"The socket already grants it."** ADR 0026 §2.3 confines the socket to known-target envelopes; a mount is a scope of its own (withdrawn in 6582932837).
- **`unknown` whenever the two counts disagree.** Their disagreement is the signal; that wording would have suppressed every diagnosis (Emmy's correction).
- **Reusing `buildStoreBloatDiagnosis`.** It measures store size for one service, not dead elements per collection.

## Related

- Consumer: #1432. Parent: #1416. Prerequisite: #1433.
- Design: #1432 comments 6582630907, 6582721976 (Ada), 6582788745, 6582932837; D#19014 (Emmy's 2026-09-20 gatherer trace).
- ADR 0019, ADR 0026, ADR 0027.

**Sweeps, 2026-11-02 17:36Z:**
- Live latest-open + closed sweep: checked the latest 20 open issues, an open search (`chroma-data mount`, `dead elements`) and a closed search (`HNSW dead elements`). Same-surface hits #60, #306, #66 and #54 are design origin or neighbors; no equivalent and no prior exclusion.
- A2A in-flight sweep: no claim on this scope since the #1432 convergence.
- MC sweep: `KB writes lost during collection rebuild, kb-server ingestion during defrag, dead HNSW elements counted on cloud`, 6 results; no prior decision.
- Own-assignment sweep: 15 open; #1432 and #1433 are this leaf's consumer and prerequisite, none overlaps.

Ownership: self-assigned as #1416's owner.

Retrieval Hint: "KB dead HNSW elements read-only chroma-data mount header.bin segment registry observation scope"

## Timeline

- 2026-11-02T17:37:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T17:37:31Z @neo-opus-vega added the `enhancement` label
- 2026-11-02T17:37:31Z @neo-opus-vega added the `ai` label
- 2026-11-02T17:37:31Z @neo-opus-vega added the `architecture` label
- 2026-11-02T17:37:32Z @neo-opus-vega added the `performance` label
- 2026-11-02T17:37:32Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T17:38:14Z @neo-opus-vega cross-referenced by #1435
- 2026-11-02T17:38:27Z @neo-opus-vega added parent issue #1416
- 2026-11-02T17:38:33Z @neo-opus-vega marked this issue as being blocked by #1433
- 2026-11-02T17:38:35Z @neo-opus-vega marked this issue as blocking #1432
- 2026-11-02T17:39:50Z @neo-opus-vega cross-referenced by #1432
- 2026-11-02T17:46:07Z @neo-opus-vega cross-referenced by PR #1436
- 2026-11-02T17:48:26Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T18:10:09Z @neo-opus-vega cross-referenced by #1433
- 2026-11-02T18:16:52Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T19:41:48Z @neo-gemini-maryam cross-referenced by #306
- 2026-11-05T10:35:10Z @neo-opus-vega cross-referenced by #1438
- 2026-11-05T11:16:41Z @neo-opus-vega cross-referenced by #66
- 2026-11-05T11:17:31Z @neo-opus-vega cross-referenced by #55
- 2026-11-05T11:18:21Z @neo-opus-vega cross-referenced by #60
- 2026-11-05T11:35:02Z @neo-opus-vega cross-referenced by PR #1478
- 2026-11-13T04:51:37Z @neo-opus-vega cross-referenced by #1674
- 2026-11-16T08:24:52Z @neo-opus-vega cross-referenced by #1795
- 2026-11-18T15:38:31Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-23T06:33:35Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-23T06:33:35Z @neo-gemini-maryam unassigned from @neo-opus-vega
### @neo-gemini-maryam - 2026-11-23T06:33:36Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-vega`
**New assignees:** `@me`
**Reason:** Accepted @neo-opus-vega handoff offer MESSAGE:95b7f3ac-0102-483c-b09a-8973344d5ac0

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

### @neo-gemini-maryam - 2026-11-23T06:33:48Z

## Ticket Intake & Structural Pre-Flight (`dev@84e2b1fb`)

Picking up the 3-leaf KB defrag chain (`#1434` → `#1435` → `#1432`) per `@neo-opus-vega`'s handoff (`MESSAGE:95b7f3ac-0102-483c-b09a-8973344d5ac0`), starting with `#1434`.

### §0 Intent & Prescription Check
- **Premise verified at `dev@84e2b1fb`:**
  - `buildStoreBloatDiagnosis` (`ai/daemons/orchestrator/services/storeBloatDiagnosis.mjs:36`) has zero production callers and keys store-size bloat to `serviceId` (`mc-server`).
  - `assembleDataIntegrityEvidence` (`ai/daemons/orchestrator/services/dataIntegrityEvidenceAssembler.mjs:70-92`) folds every `store-bloat` fact into the single `serviceId` store row instead of a collection-scoped row when `fact.collection` is present.
  - `deploy/cloud/docker-compose.yml` mounts `chroma-data` only into `chroma:/data` (`l.36`); `orchestrator` (`l.530-593`) does not mount `chroma-data`, while `AiConfig.engines.chroma.dataDir` resolves to `/app/.neo-ai-data/chroma/unified` (`ai/configBase.mjs:1325, 3028`).
  - `Orchestrator.dataIntegrityEvidenceGatherer` (`ai/daemons/orchestrator/Orchestrator.mjs:1525-1541`) awaits `dataIntegrityCoverageGatherer` (`sqlite3 -json`) unconditionally before any sibling diagnosis; when `sqlite3` is absent in the container image, the throw blanks the entire gather into top-level `probe-unavailable`.
  - Prerequisite `#1433` is closed (`92a7ec44`, PR `#1436`), so `data-integrity-sweep` now carries its `TaskStateService` envelope.
- **Gatherer isolation falsifier (`Fix step 4`):** Checked `dataIntegrityModeClassifier.mjs` (`l.81-129`), `Orchestrator.mjs` (`l.1512`), and `dimensionConsistencyGatherer.mjs` (`l.80-81`). Neither coverage nor dimension gatherers touch `neo-knowledge-base`, and `classifyDataIntegrityMode` evaluates each collection row independently. Isolating coverage failures to `probes.coverage = {status: 'probe-unavailable', probeError}` emits zero coverage rows (preventing both coverage heals and `liftQuarantine` on unprobed MC collections) while letting the KB dead-element diagnosis reach `assembleDataIntegrityEvidence` → `classifyDataIntegrityMode` (`defrag`).
- **Source-verified Chroma `1.5.9` binary & Version Witness:**
  - Verified against `chroma-core/hnswlib@6868102bde454dc761136e1994490133a6a026bb` (`hnswlib/hnswalg.h:20-34, 161, 1043-1052`): `persistHeader` writes the 14 `HEADER_FIELDS` in 100 little-endian bytes (`PERSISTENCE_VERSION=1` int32 at `0..3`, `offsetLevel0_=0` uint64 at `4..11`, `max_elements_` uint64 at `12..19`, `cur_element_count` uint64 at `20..27`, `size_data_per_element_` uint64 at `28..35`, `label_offset_` uint64 at `36..43`, `offsetData_` uint64 at `44..51`, `maxlevel_` int32 at `52..55`, `enterpoint_node_` uint32 at `56..59`, `maxM_` uint64 at `60..67`, `maxM0_` uint64 at `68..75`, `M_` uint64 at `76..83`, `mult_` float64 at `84..91`, `ef_construction_` uint64 at `92..99`), cross-checked against `collections.dimension` (`offsetData === 4 + maxM0*4`, `labelOffset === offsetData + dim*4`, `sizeDataPerElement === labelOffset + 8`).
  - Per Emmy's `D#19014` (`DC_kwDODSospM4BGpyb`) finding, Chroma `1.5.9`'s HTTP version endpoint reports crate version `1.0.0`, so Chroma `1.5.9` version admission binds to the runtime-inspected container image (`chromadb/chroma:1.5.9` ± digest), SQLite `migrations` (`sysdb=10`, `metadb=6`, `embeddings_queue=2` when present), and `header.bin` `PERSISTENCE_VERSION === 1`.
  - Per Emmy's `D#19014` (`DC_kwDODSospM4BGp7o`, `DC_kwDODSospM4BGp2-`) checkpoint/retained-log boundary, `liveCount > hnswOccupancy + postCheckpointLogCount` (from `max_seq_id` + `embeddings_queue`) is classified as `unknown` (incoherent checkpoint/empty-materialized index), whereas `hnswOccupancy > liveCount` computes `(hnswOccupancy - liveCount) / hnswOccupancy` against `AiConfig.knowledgeBase.gcDefragThreshold`.
- **Prescription checked:** `ai/daemons/orchestrator/services/kbDeadElementDiagnosis.mjs` + `ai/daemons/orchestrator/services/kbDeadElementGatherer.mjs` + `ai/daemons/orchestrator/services/dataIntegrityEvidenceAssembler.mjs` + `ai/daemons/orchestrator/Orchestrator.mjs` — owns the concern.
- **Core-Idiom Pre-Flight (§1.9.6):** Read `src/core/Base.mjs`, `src/Neo.mjs`, and `src/state/Provider.mjs`; `Orchestrator.mjs` and `DataIntegrityDiagnosisService.mjs` keep collaborator seams as set-once plain fields/getters and read `AiConfig` leaves fresh inside the gatherer closure (ADR-0019 §3: zero `?.`, zero `B2` primitive capture, zero `B5` threading, zero `C1` non-entrypoint `AiConfig` imports).

### §1.8 Ticket Reality & ADR Audit
- `Ticket age`: `createdAt: 2026-11-02T17:37:29Z`, `updatedAt: 2026-11-02T17:37:29Z`.
- `Bot stale-band`: `pre-stale` (`stale: false`, `no auto close: false`).
- `Currency / successor-risk evidence`: Checked `git log --since=2026-11-02T17:37:29Z` across `deploy/cloud/`, `ai/daemons/orchestrator/`, `learn/agentos/decisions/`, and `ai/scripts/maintenance/`. `#1436` (`92a7ec44`) closed prerequisite `#1433`; `#1473` (`65182b08`) added `ProbeMutationDispositionGuard.spec.mjs` (`ABORT_DEADLINE_MODULE_CENSUS`); `#1478` (`b282d7eb`) documented compose memory caps; `#1758` (`d6e65d75`) touched disk-pressure prune. Open PR queue (`#1816`) has zero overlap; Memory Core sweep (`e6bcc40d`, `f983fdbd`, `a7a48a8a`) and KB sweep (`#133`, `#1416`, `#60`, `D#19014`) confirm no duplicate or superseding producer.
- `ADR successor-risk`: `adr-amendment-required` — artifact `#1434/2026-11-02`; ADR `neomjs/neo-agent-brain#adr:recovery-actuator` (`Accepted/2026-06-23`, last amended `2026-11-02` via `#1430`); evidence `learn/agentos/decisions/0026-recovery-actuator.md` §2.3 + `learn/agentos/decisions/0027-autonomous-data-recovery-actuator.md` §2.5 + `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md`; route `amendment-required` (shipped in AC-1 of this PR).
- `Verdict`: `valid-as-written` (positive ROI — unblocks cloud KB compaction `#1432` / `#1416` AC-3 without full-DB copies).

### Structural Pre-Flight (§2 Full)
> *"Pre-Flight (structural full): considered destinations `ai/daemons/orchestrator/services/`, `ai/scripts/maintenance/`, and `ai/scripts/diagnostics/` for `kbDeadElementDiagnosis.mjs` and `kbDeadElementGatherer.mjs`; consulted the Engine [Architecture Overview](https://github.com/neomjs/neo/blob/dev/learn/benefits/ArchitectureOverview.md) Structural Inventory (`ai/daemons/` row covers orchestrator recovery and data-integrity services; `ai/scripts/` is one-shot CLI scripts under `scriptPlaneClosure.mjs`), canonical ADRs (`neomjs/neo-agent-brain#adr:recovery-actuator` §2.3/§2.7 confining the `:ro` `chroma-data` store-observation scope to the orchestrator's dead-element reader alone; `neomjs/neo-agent-brain#adr:autonomous-data-recovery-actuator` §2.5 `store-bloat → defrag`), and `learn/agentos/v13-path.md` §D3 (`ai/daemons/orchestrator/services/` hosts pure diagnosis producers `storeBloatDiagnosis.mjs` / `dimensionConsistencyDiagnosis.mjs` alongside live gatherers `dimensionConsistencyGatherer.mjs` / `bootIdentityFactGatherer.mjs`); chose `ai/daemons/orchestrator/services/kbDeadElementDiagnosis.mjs` (pure producer + 100-byte header validator) and `ai/daemons/orchestrator/services/kbDeadElementGatherer.mjs` (read-only SQLite registry + `header.bin` + runtime-identity gatherer) because they share the exact producer+gatherer role of `dimensionConsistencyDiagnosis.mjs` + `dimensionConsistencyGatherer.mjs` and keep the ADR-0026 `:ro` store-observation reader inside the orchestrator boundary rather than exposing it as an `ai/scripts/` CLI helper. Chief-architect framing: (1) Scalability — matches the 6 existing `*Diagnosis.mjs` + `*Gatherer.mjs` leaves in `orchestrator/services/`; (2) ADR-conflict — none once ADR 0026 §2.3/§2.10 + AC-15 declare the `:ro` store-observation scope in AC-1; (3) ADR-genesis — covered by the ADR 0026 amendment in this PR; (4) Future-self regression-risk — exporting `readKbDeadElementObservation` cleanly supplies `#1432` Fix step 2's op-start re-read without coupling to `Orchestrator.mjs`. Map-maintenance: `not-needed` (`ArchitectureOverview.md` `ai/daemons/` row already covers orchestrator data-integrity services and links ADRs 0025/0026/0027)."*

🧭 **Maryam** · `@neo-gemini-maryam` · Gemini 4 Carbon · Antigravity

- 2026-11-23T07:00:58Z @neo-gemini-maryam cross-referenced by PR #1908
- 2026-11-23T08:25:06Z @neo-gemini-maryam referenced in commit `7d2b079` - "fix(orchestrator): read KB dead-element config at gatherer use site (#1434)"
- 2026-11-23T08:58:02Z @neo-gemini-maryam cross-referenced by PR #1909
- 2026-11-23T12:18:47Z @tobiu referenced in commit `42c211e` - "feat(orchestrator): measure KB dead HNSW elements on cloud plane (#1434) (#1908)

* feat(orchestrator): measure KB dead HNSW elements on cloud plane (#1434)

* fix(orchestrator): read KB dead-element config at gatherer use site (#1434)"
- 2026-11-23T12:18:49Z @tobiu closed this issue
- 2026-11-23T18:59:41Z @neo-gemini-maryam cross-referenced by PR #1912

