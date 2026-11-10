---
id: 1548
title: Guard authoritative CONCEPT nodes in GraphService.upsertNode from REM clobber
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-11-09T22:00:21Z'
updatedAt: '2026-11-09T22:59:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1548'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 1258
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
# Guard authoritative CONCEPT nodes in GraphService.upsertNode from REM clobber

## Context

Design authority: `learn/agentos/ConceptOntology.md`, `ai/services/memory-core/GraphService.mjs:398-458` (`preserveAdrAuthority` precedent), and #1258 (`IC_kwDOUBzDFM8AAAABknLidA` item 3: *"One AC to add there: a census of concept ids whose graph node is not typed `CONCEPT`, before and after. If that is more than a handful, the writer side gets its own ticket"*).

In #1547 (Leaf 3 of #1258), the pre-fix census of all `405` concept IDs in `concepts/nodes.jsonl` (`59` curated with `validated !== false`, `346` mined candidates with `validated: false`) against the live Memory Core SQLite `Nodes` table (`2026-11-09T21:44Z`) found:
- **`7` ontology nodes whose SQLite `label` was overwritten from `'CONCEPT'` to `'CLASS'` (`6` of `59` curated concepts [10.2%] + `1` candidate):** `neo-mjs`, `delta-updates`, `agent-os`, `knowledge-base`, `grid-component`, `tree-store`, and `seat-model`.
- **`42` ontology nodes whose `name` or `description` was overwritten (`17` of `59` curated concepts [28.8%] + `25` candidates):** `neo-mjs`, `worker-isolation`, `delta-updates`, `setup-class`, `instance-lifecycle`, `object-permanence`, `agent-os`, `neural-link`, `knowledge-base`, `memory-core`, `dream-pipeline`, `swarm-intelligence`, `golden-path`, `state-provider`, `theming-engine`, `grid-component`, and `tree-store`.
- **`59` ontology nodes carrying foreign `summary_*` `semanticVectorId` values** written by `SemanticGraphExtractor.commitTriVectorPayload`.

`unowned-rationale: Ready for a contributor or maintainer pickup after #1547 lands; maintainers provide planning and review.`

---

## The Problem

While #1547 repairs clobbered `CONCEPT` nodes during `ConceptIngestor.syncConceptsToGraph()`, **intra-cycle writer ordering in `RemDigestion.processUndigestedSessions()` (`src/evolution/RemDigestion.mjs`)** still allows `SemanticGraphExtractor` to clobber projected `CONCEPT` nodes mid-cycle before Phase 4 gap inference runs:

1. **Phase 0b (`src/evolution/RemDigestion.mjs:548`):** `await this.conceptIngestor.syncConceptsToGraph()` upserts ontology `CONCEPT` nodes with `properties.payloadHash` (64-char hex sha256) and `properties.conceptProjectionVersion`.
2. **Per-Session Loop (`src/evolution/RemDigestion.mjs:718`):** `await this.semanticGraphExtractor.executeTriVectorExtraction(session)` runs for each undigested session. `SemanticGraphExtractor.commitTriVectorPayload` (`ai/services/graph/SemanticGraphExtractor.mjs:856`) calls `canonicalizeSemanticGraphNodeId({id: nodeId, type: nodeType, name: node.name})` (`ai/services/graph/conceptSpineCanonicalization.mjs:16-29`), which strips `CLASS:` / `PROCESS:` / `CONCEPT:` prefixes and lower-kebabs `node.name`. Whenever an LLM-extracted `CLASS` or `CONCEPT` entity shares a kebab ID with a projected ontology node (e.g., `CLASS` `"Neo Agent OS Memory Management"` -> `agent-os`, `CLASS` `"Neo.mjs Platform"` -> `neo-mjs`, `CLASS` `"TreeStore.mjs"` -> `tree-store`), `GraphService.upsertNode` overwrites `label` to `'CLASS'`, `name`, `description`, and `semanticVectorId` to `'summary_...'`.
3. **Phase 4 (`src/evolution/RemDigestion.mjs:937`):** `await this.inferConceptGraphGaps()` and `inferTestGapsFromSession` run on the already-clobbered nodes within the **same** REM cycle — hiding `agent-os` from `GapInferenceEngine.inferConceptGraphGaps()` (`n.label === 'CONCEPT'` filter at `GapInferenceEngine.mjs:280`) and emitting false `[TEST_GAP]` alerts for `neo-mjs` (`"The CLASS 'Neo.mjs Platform' lacks corresponding automated validation suites"`) and `tree-store`.

---

## The Architectural Reality

- `ai/services/memory-core/GraphService.mjs` (`lines 398-458`): already implements `preserveAdrAuthority` for admitted `ADR` nodes (`existingLabel === 'ADR' && existingProps.adrRecordKind === 'decision-record' && incomingProps.adrRecordKind !== 'decision-record'`), preserving `label: 'ADR'`, `existingProps.name`, `existingProps.description`, and `existingProps.semanticVectorId` when `SemanticGraphExtractor` or another non-authoritative writer upserts a colliding node ID.
- `ai/services/ingestion/ConceptIngestor.mjs` (`lines 580-690`): stamps `payloadHash` (`64`-char sha256 hex) and `conceptProjectionVersion` on every ontology-backed `CONCEPT` node (note from `PR #1549` review `5833965544` RA-2: do not key on a separate `sourceOfTruth` flag outside `computePayloadHash`, because unchanged nodes skipped by differential sync would lack it; `existingLabel === 'CONCEPT' && /^[a-f0-9]{64}$/.test(existingProps.payloadHash || '')` covers all projected ontology nodes).
- **Pre-implementation V-B-A question (`PR #1549` review `5833965544` Challenge 1):** Verify whether any reader hydrates a `CONCEPT` node's `semanticVectorId` (e.g. `ai/services/memory-core/helpers/conceptAnchoredRetrieval.mjs:10-12`) before deciding whether `preserveConceptAuthority` should keep `semanticVectorId: null` or allow/manage vector binding on shared spine IDs.

---

## The Fix

1. **Add `preserveConceptAuthority` in `GraphService.upsertNode` (`ai/services/memory-core/GraphService.mjs:398-458`):**
   - Alongside `preserveAdrAuthority`, detect when an existing node is an ontology-projected `CONCEPT` node (`existingLabel === 'CONCEPT' && /^[a-f0-9]{64}$/.test(existingProps.payloadHash || '')`) and the incoming upsert is not a `ConceptIngestor` ontology projection (`!/^[a-f0-9]{64}$/.test(incomingProps.payloadHash || '')` and `incomingProps.conceptProjectionVersion == null`).
   - When `preserveConceptAuthority` is `true`:
     - Preserve `label: 'CONCEPT'` (preventing `CLASS` / `PROCESS` / `architectural_decision` overwrites).
     - Preserve `existingProps.name` and `existingProps.description`.
     - Preserve `existingProps.semanticVectorId` according to the Challenge 1 V-B-A check on `conceptAnchoredRetrieval.mjs`.
2. **Unit Tests (`test/playwright/unit/ai/services/memory-core/GraphService.spec.mjs` or `ConceptIngestor.spec.mjs`):**
   - Prove that after `ConceptIngestor.syncConceptsToGraph()` seeds an ontology `CONCEPT` node (`payloadHash`), a subsequent `GraphService.upsertNode({id: 'agent-os', type: 'CLASS', name: 'Neo Agent OS Memory Management', description: '...', semanticVectorId: 'summary_123'})` without `payloadHash` preserves `label: 'CONCEPT'`, canonical `name`, and canonical `description`.
   - Prove that `ConceptIngestor` (`properties: {payloadHash, ...}`) and `ConceptIngestor.updateProjectionMetadata` (`properties: {conceptProjectionVersion, ...}`) continue to update the node when the ontology row changes.

---

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `GraphService.upsertNode` `preserveConceptAuthority` guard (`ai/services/memory-core/GraphService.mjs`) | `preserveAdrAuthority` (`GraphService.mjs:398-458`); #1258; #1547; `PR #1549` review `5833965544` | When `existingLabel === 'CONCEPT'` with a valid 64-hex `existingProps.payloadHash` and the incoming write carries neither `payloadHash` nor `conceptProjectionVersion`, preserve `label: 'CONCEPT'`, `name`, `description`, and `semanticVectorId` | Authoritative `ConceptIngestor` upserts (`payloadHash` / `conceptProjectionVersion`) update normally | JSDoc on `GraphService#upsertNode` | Unit test arms for non-authoritative `CLASS`/`summary_*` collision vs. authoritative ontology update |

Decision Record impact: aligned-with ADR 0023, ADR 0024, ADR 0033 (none amended).

---

## Acceptance Criteria

- [ ] AC-1 When an existing graph node has `label === 'CONCEPT'` and a 64-char hex `properties.payloadHash`, a `GraphService.upsertNode` call without `payloadHash` / `conceptProjectionVersion` (such as `SemanticGraphExtractor.commitTriVectorPayload`) cannot overwrite its `label`, `properties.name`, `properties.description`, or `properties.semanticVectorId`.
- [ ] AC-2 Authoritative `ConceptIngestor.syncConceptsToGraph()` and `updateProjectionMetadata()` upserts continue to update `name`, `description`, `tier`, `weight`, `payloadHash`, and `conceptProjectionVersion` when the ontology changes.
- [ ] AC-3 Unit tests cover both the rejected non-authoritative overwrite (`type: 'CLASS'`, `semanticVectorId: 'summary_*'`) and the permitted authoritative ontology update.

---

## Out of Scope

- Read/repair-side `ConceptIngestor.syncConceptsToGraph()` drift detection (covered by #1547 / PR #1549).
- Changing `canonicalizeSemanticGraphNodeId` kebab-case normalization rules in `conceptSpineCanonicalization.mjs`.

---

## Avoided Traps

- **Relying solely on Phase 0b `ConceptIngestor.syncConceptsToGraph()` repair (#1547) without a writer guard:** because `RemDigestion` runs Phase 0b *before* per-session `SemanticGraphExtractor` extraction (line 718) and runs Phase 4 gap inference *after* extraction (line 937), any concept clobbered during the session loop remains clobbered during Phase 4 gap inference until the *next* REM cycle's Phase 0b.
- **Keying on a field outside `computePayloadHash` (`PR #1549` review `5833965544` RA-2):** any property not included in `computePayloadHash` is only written on re-upserted nodes and missed on skipped unchanged nodes; `payloadHash` (`64`-char sha256 hex) is already present on every projected ontology node.

---

## Related

- Parent issue: #1258 (`IC_kwDOUBzDFM8AAAABknLidA` item 3)
- Sibling read/repair leaf: #1547 (PR #1549)
- Precedent (`preserveAdrAuthority`): `ai/services/memory-core/GraphService.mjs:398-458`

Live latest-open + closed sweep: checked latest 20 open (`167` open total — red band > 50; split required by #1258 `IC_kwDOUBzDFM8AAAABknLidA` item 3) + closed search (`ConceptIngestor agent-os orphan concepts edges CLASS`, `0` closed hits) at `2026-11-09T21:59Z`; no equivalent found (#1547 covers the ingestor repair). A2A in-flight claim sweep: `list_messages({status: 'all', limit: 30})` at `2026-11-09T21:59Z`, `0` overlapping claims. MC sweep: `query_raw_memories("concept node overwritten to CLASS by REM extraction payloadHash skip")`, `6` results. Own-assignment sweep: `2` open in Brain (`#1230`, `#1547`), `#1547` is the sibling read/repair leaf. Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/memory-core/GraphService.mjs` (`2342` LOC); no new file.

Origin Session ID: 132c8467-16c8-4c19-ab83-24c483477a21

Retrieval Hint: "preserveConceptAuthority GraphService upsertNode payloadHash SemanticGraphExtractor CLASS clobber summary_ semanticVectorId #1258 #1547"


## Timeline

- 2026-11-09T22:00:23Z @neo-gemini-julia added the `bug` label
- 2026-11-09T22:00:23Z @neo-gemini-julia added the `good first issue` label
- 2026-11-09T22:00:24Z @neo-gemini-julia added the `help wanted` label
- 2026-11-09T22:00:24Z @neo-gemini-julia added the `ai` label
- 2026-11-09T22:00:24Z @neo-gemini-julia added the `architecture` label
- 2026-11-09T22:00:24Z @neo-gemini-julia added the `agent-os` label
- 2026-11-09T22:01:05Z @neo-gemini-julia added parent issue #1258
- 2026-11-09T22:11:24Z @neo-gemini-julia cross-referenced by PR #1549
- 2026-11-09T22:55:20Z @neo-fable-mnemosyne cross-referenced by #1551
- 2026-11-09T22:59:12Z @neo-gemini-julia cross-referenced by #1547

