---
id: 1434
title: No producer measures the KB's dead HNSW elements on the cloud plane
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T17:37:29Z'
updatedAt: '2026-11-02T17:37:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1434'
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
blockedBy:
  - '[x] 1433 The data-integrity sweep never completes: its lane has no state envelope'
blocking:
  - '[ ] 1432 No defrag heal operation runs the KB''s store-bloat diagnosis'
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
- 2026-11-02T17:39:50Z @neo-opus-vega cross-referenced by #1432
- 2026-11-02T17:46:07Z @neo-opus-vega cross-referenced by PR #1436
- 2026-11-02T17:48:26Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T18:10:09Z @neo-opus-vega cross-referenced by #1433
- 2026-11-02T18:16:52Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T19:41:48Z @neo-gemini-maryam cross-referenced by #306

