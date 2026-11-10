---
id: 1547
title: Repair CONCEPT nodes clobbered by REM extraction in ConceptIngestor.syncConceptsToGraph
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-09T21:59:24Z'
updatedAt: '2026-11-09T23:14:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1547'
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
closedAt: '2026-11-09T23:14:29Z'
---
# Repair CONCEPT nodes clobbered by REM extraction in ConceptIngestor.syncConceptsToGraph

## Context

Design authority: `AGENTS.md §neo_identity_anchor`, `learn/agentos/ConceptOntology.md`, ADR 0024 §2.6 (`#472`), `#471`, and parent issue #1258 (`IC_kwDOUBzDFM8AAAABknHGBg` / `IC_kwDOUBzDFM8AAAABknLidA`).

Parent ticket #1258 (*"A premise check against the identity anchor before positioning work"*) includes repairing `ConceptIngestor.syncConceptsToGraph()` so Concept Ontology nodes overwritten by REM session-summary extraction are detected and restored to `label: 'CONCEPT'`, alongside the pre/post SQLite census and the companion writer-guard leaf (`#1548`). Per ADR 0024 §2.6 (`#472`) and `#471` (`#1549` Round-1 review `5833965544` RA-1), plane ontology edge rows enter the plane store as data under a separate `#1258` leaf rather than residing in Brain source.

Excluded from `good first issue`: already-owned sub-issue under #1258 (`@neo-gemini-julia`) requiring live plane ontology verification and pre/post SQLite graph census receipts.

### Pre-Fix Ontology vs. SQLite Graph Census (`2026-11-09T21:44Z`)

- **Command / Method:** Read-only comparison of plane ontology `concepts/nodes.jsonl` (`405` rows: `59` curated with `validated !== false`, `346` mined candidates with `validated: false`) and `concepts/edges.jsonl` (`182` rows) against the live Memory Core SQLite `Nodes` table (`SELECT id, data FROM Nodes WHERE id = ?`).
- **Scope:** All `405` concept IDs in `concepts/nodes.jsonl`.
- **Result (`nonConceptCount: 7`, `driftedNameOrDescCount: 42`, `summaryVectorCount: 59`, `missingCount: 2`):**
  1. **7 concept IDs whose graph node is NOT typed `CONCEPT` (`6` of `59` curated concepts + `1` candidate, all overwritten to `label: 'CLASS'` by REM `SemanticGraphExtractor`):**
     - `neo-mjs` (tier 0, curated): `label: 'CLASS'`, `graphName: 'Neo.mjs Platform'` (ontology: `'Neo.mjs'`), `semanticVectorId: 'summary_c5b059c4-5d96-46f1-8c61-184af74d34b4'` — caused `neo-mjs` to surface as the #1 `[TEST_GAP]` in `sandman_handoff.md`.
     - `delta-updates` (tier 1, curated): `label: 'CLASS'`, `graphName: 'DeltaUpdates Engine'` (ontology: `'Delta Update Pipeline'`), `semanticVectorId: 'summary_689823ba-dd56-4db2-b2bf-933ef1fffe2d'`.
     - `agent-os` (tier 1, curated): `label: 'CLASS'`, `graphName: 'Neo Agent OS Memory Management'` (ontology: `'AI-Native Architecture (Agent OS)'`), `semanticVectorId: 'summary_65ddc71f-04c4-4c7d-90f0-126e11f23681'` — hid `agent-os` from `GapInferenceEngine.inferConceptGraphGaps()` (`n.label === 'CONCEPT'` filter at `GapInferenceEngine.mjs:280`).
     - `knowledge-base` (tier 2, curated): `label: 'CLASS'`, `graphName: 'Knowledge Base'` (ontology: `'Knowledge Base (Semantic RAG)'`), `semanticVectorId: 'summary_3055d81f-f3c2-4633-90c7-f66617fbacd8'`.
     - `grid-component` (tier 2, curated): `label: 'CLASS'`, `graphName: 'Grid Component'`, `semanticVectorId: 'summary_724f82dc-7b30-4e9d-a9e3-ed40ed58fa52'`.
     - `tree-store` (tier 3, curated): `label: 'CLASS'`, `graphName: 'TreeStore.mjs'` (ontology: `'TreeStore & Hierarchical Data'`), `semanticVectorId: 'summary_a2ea9e87-f331-4002-93d1-f66617fbacd8'` — caused `tree-store` to surface as the #3 `[TEST_GAP]` in `sandman_handoff.md`.
     - `seat-model` (tier 3, candidate `validated: false`): `label: 'CLASS'`, `graphName: 'SeatModel'` (ontology: `'Seat Model'`), `semanticVectorId: 'summary_5cf68289-98e2-4e7e-9dff-42c250e5700a'`.
  2. **42 concept IDs with clobbered `name` or `description` (`17` of `59` curated concepts [28.8%] + `25` candidates):**
     - Curated concepts (`17`): `neo-mjs`, `worker-isolation`, `delta-updates`, `setup-class`, `instance-lifecycle`, `object-permanence`, `agent-os`, `neural-link`, `knowledge-base`, `memory-core`, `dream-pipeline`, `swarm-intelligence`, `golden-path`, `state-provider`, `theming-engine`, `grid-component`, `tree-store`.
  3. **59 concept IDs carrying foreign `summary_*` `semanticVectorId` values** written by `SemanticGraphExtractor.commitTriVectorPayload`.

Because `7` label-clobbered and `42` name/description-clobbered ontology nodes exceed the "more than a handful" threshold (`#1258` `IC_kwDOUBzDFM8AAAABknLidA` item 3), the writer-side authority guard (`GraphService.upsertNode` / `SemanticGraphExtractor`) is filed as sibling ticket #1548 alongside this leaf.

---

## The Problem

**`ConceptIngestor.syncConceptsToGraph()` skips clobbered `CONCEPT` nodes (`ai/services/ingestion/ConceptIngestor.mjs:667`):**
- When `SemanticGraphExtractor` extracts a session-summary entity whose canonicalized kebab ID (`conceptSpineCanonicalization.mjs:16-29`) collides with a Concept Ontology ID (e.g., `agent-os`, `neo-mjs`, `delta-updates`), `GraphService.upsertNode` overwrites `node.label` (`'CONCEPT'` -> `'CLASS'`), `node.properties.name`, `node.properties.description`, and `node.properties.semanticVectorId` (`'summary_...'`), while preserving `node.properties.payloadHash` via shallow property merge.
- On subsequent `ConceptIngestor.syncConceptsToGraph()` passes (`ConceptIngestor.mjs:667`), `nodePayloadMatch = existingNode?.properties?.payloadHash === payloadHash` still evaluates `true`, so `ConceptIngestor` increments `stats.conceptsSkipped` and never restores `label: 'CONCEPT'`, `name`, `description`, or clears the foreign `summary_*` `semanticVectorId`.
- Even when `ConceptIngestor` does call `GraphService.upsertNode` (`ConceptIngestor.mjs:672-690`), it omits `semanticVectorId: null`, so `GraphService.upsertNode` (`GraphService.mjs:427-436`) preserves the foreign `summary_*` `semanticVectorId`.

---

## The Architectural Reality

- `ai/services/ingestion/ConceptIngestor.mjs` (`lines 580-605`, `658-695`): projects `ConceptService` nodes and edges into `GraphService`. In `GraphService.db.nodes.get(conceptId)`, the node's type is read via `existingNode.isRecord ? existingNode.get('label') : existingNode.label` (`GraphService.mjs:395`), and `name`, `description`, and `semanticVectorId` live on `existingNode.properties`.
- `GraphService.upsertNode` (`ai/services/memory-core/GraphService.mjs:427-436`): preserves `existingProps.semanticVectorId` unless `semanticVectorId !== undefined` is explicitly passed (`semanticVectorId: null` clears a foreign `summary_*` vector on a concept node that does not own a summary vector).
- `src/evolution/RemDigestion.mjs` (`line 548`): Phase 0b runs `await this.conceptIngestor.syncConceptsToGraph()`.

---

## The Fix

1. **Harden `ConceptIngestor.syncConceptsToGraph()` & `updateProjectionMetadata()` (`ai/services/ingestion/ConceptIngestor.mjs:580-605, 658-695`):**
   - Check `existingLabel === 'CONCEPT'` (`existingNode ? (existingNode.isRecord ? existingNode.get('label') : existingNode.label) : null`), `existingProps.name === conceptNode.name`, `(existingProps.description ?? '') === (conceptNode.description ?? '')`, and `!String(existingProps.semanticVectorId || existingNode?.semanticVectorId || '').startsWith('summary_')` alongside `existingProps.payloadHash === payloadHash` before taking the `conceptsSkipped` fast path (and require `existingLabel === 'CONCEPT'` in `updateProjectionMetadata`).
   - When `hasSummaryVector` is `true`, pass `semanticVectorId: null` to `GraphService.upsertNode` so the foreign `summary_*` vector reference is cleared.
2. **Unit Tests (`test/playwright/unit/ai/services/ingestion/ConceptIngestor.spec.mjs`):**
   - Add unit test proving that when a `CONCEPT` node (`agent-os`) is clobbered to `type: 'CLASS'` with a foreign `name`, `description`, and `semanticVectorId: 'summary_...'` while retaining its `payloadHash`, `ConceptIngestor.syncConceptsToGraph()` detects the drift, re-upserts `label: 'CONCEPT'`, restores canonical `name`/`description`, clears `semanticVectorId: null`, and skips on the subsequent third pass (`conceptsSkipped === 1`).

---

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `ConceptIngestor.syncConceptsToGraph` & `updateProjectionMetadata` node-skip predicate (`ai/services/ingestion/ConceptIngestor.mjs`) | `GraphService.mjs:395, 427-436`; #1258 | Requires `label === 'CONCEPT'`, matching `name` & `description`, `!semanticVectorId.startsWith('summary_')`, and matching `payloadHash` before skipping; clears `semanticVectorId: null` when repairing `summary_*` | Preserves non-`summary_*` `semanticVectorId` when present; third pass takes `conceptsSkipped` fast path | JSDoc on `syncConceptsToGraph` | `ConceptIngestor.spec.mjs` clobber-repair + skip-recovery arm + live pre/post census |

Decision Record impact: aligned-with ADR 0023, ADR 0024, ADR 0033 (none amended).

---

## Acceptance Criteria

- [ ] AC-1 `ConceptIngestor.syncConceptsToGraph()` detects when an ontology node in `GraphService` has `label !== 'CONCEPT'`, drifted `name` or `description`, or a `summary_*` `semanticVectorId` even when `properties.payloadHash` matches, restores `label: 'CONCEPT'` + canonical `name`/`description`, clears `semanticVectorId: null`, and skips on the subsequent pass (`conceptsSkipped === 1` in `ConceptIngestor.spec.mjs`).
- [ ] AC-2 Pre-and-post census of `concepts/nodes.jsonl` concept IDs whose SQLite graph node is not typed `CONCEPT` (or has drifted `name`/`description` or `summary_*` vector): pre-fix census recorded above (`7` non-`CONCEPT`, `42` drifted `name`/`description`, `59` `summary_*`); post-fix sync reduces all three counts to `0` across all synced ontology nodes, and the writer-side authority guard ticket (#1548) is filed for the intra-cycle `SemanticGraphExtractor` overwrite.

---

## Out of Scope

- Writer-side authority guard in `GraphService.upsertNode` / `SemanticGraphExtractor` preventing intra-cycle session-summary extraction from overwriting `ConceptIngestor`-owned `CONCEPT` nodes between Phase 0b and Phase 4 (filed as sibling ticket #1548 per #1258 `IC_kwDOUBzDFM8AAAABknLidA` item 3).
- Plane-data entry path for the five identity concepts' (`json-first-vdom`, `agent-os`, `multi-window`, `multi-threading`, `data-layer`) curated edges (scoped out of Brain source per ADR 0024 §2.6 / `#471` / `#472` and `PR #1549` review `5833965544` RA-1; tracked under #1258 by `@neo-fable-mnemosyne`).
- The `checkPremise` helper and positioning-artifact pre-commit / corpus-lint / Focus v1 adherence probes (#1258 leaves 1–2, owned by `@neo-fable-mnemosyne`).

---

## Avoided Traps

- **Relying on `payloadHash` alone for differential node sync:** `GraphService.upsertNode` shallow-merges `existing.properties` into `node.properties`, so a foreign writer (`SemanticGraphExtractor`) that omits `payloadHash` preserves `ConceptIngestor`'s `payloadHash` while clobbering `label`, `name`, `description`, and `semanticVectorId`.
- **Hardcoding deployment-specific ontology edge rows in Brain source (`ADR 0024 §2.6` / `#471`):** Each plane's store is its single source of truth; Brain source must project whatever ontology rows the plane store holds without writing deployment-specific concept edges on every REM cycle.

---

## Related

- Parent issue: #1258 (`IC_kwDOUBzDFM8AAAABknHGBg`, `IC_kwDOUBzDFM8AAAABknLidA`)
- Writer-side authority guard: #1548
- Plane concept store custody: #471, #472, ADR 0024 §2.6

Live latest-open + closed sweep: checked latest 20 open (`166` open total — red band > 50; scoped sub-issue under #1258) + closed search (`ConceptIngestor agent-os orphan concepts edges CLASS`, `0` closed hits; read #526 and #471 bodies) at `2026-11-09T21:57Z`; no equivalent or prior exclusion found. A2A in-flight claim sweep: `list_messages({status: 'all', limit: 30})` at `2026-11-09T21:57Z`, `0` overlapping claims (`@neo-fable-mnemosyne` delegated this leaf to `@neo-gemini-julia` in `MESSAGE:dc9c964a-c29a-4aa2-af1c-82fc17c73fff`). MC sweep: `query_raw_memories("concept node overwritten to CLASS by REM extraction payloadHash skip")`, `6` results (all from our #1258 peer-read session turns). Own-assignment sweep: `1` open in Brain (`#1230`), `0` overlapping. Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/ingestion/ConceptIngestor.mjs` (`736` LOC); no new directory or `.mjs` file.

Origin Session ID: 132c8467-16c8-4c19-ab83-24c483477a21

Retrieval Hint: "ConceptIngestor syncConceptsToGraph agent-os CLASS clobber payloadHash summary_ semanticVectorId #1258 #1547"


## Timeline

- 2026-11-09T21:59:25Z @neo-gemini-julia added the `enhancement` label
- 2026-11-09T21:59:26Z @neo-gemini-julia added the `ai` label
- 2026-11-09T21:59:26Z @neo-gemini-julia added the `architecture` label
- 2026-11-09T21:59:26Z @neo-gemini-julia added the `agent-os` label
- 2026-11-09T22:00:16Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-09T22:00:23Z @neo-gemini-julia cross-referenced by #1548
- 2026-11-09T22:01:00Z @neo-gemini-julia added parent issue #1258
- 2026-11-09T22:11:24Z @neo-gemini-julia cross-referenced by PR #1549
- 2026-11-09T22:22:49Z @neo-gemini-julia referenced in commit `b2f71e3` - "fix(concepts): use inspectGraph.mjs for agent-os EXEMPLIFIED_BY edge (#1547)"
- 2026-11-09T22:48:50Z @neo-gemini-julia cross-referenced by #1550
- 2026-11-09T22:55:20Z @neo-fable-mnemosyne cross-referenced by #1551
- 2026-11-09T22:56:51Z @neo-gemini-julia referenced in commit `defecd9` - "fix(concepts): narrow #1547 to projection repair and remove sourceOfTruth (#1547)"
- 2026-11-09T22:59:11Z @neo-gemini-julia changed title from **Anchor identity concepts and guard CONCEPT nodes from REM clobber** to **Repair CONCEPT nodes clobbered by REM extraction in ConceptIngestor.syncConceptsToGraph**

