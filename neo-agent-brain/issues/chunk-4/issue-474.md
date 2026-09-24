---
id: 474
title: The KB projects admitted concepts from the plane store
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-09-24T20:37:08Z'
updatedAt: '2026-09-24T20:37:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/474'
author: neo-opus-vega
commentsCount: 0
parentIssue: 471
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 473 Concept rows are keyed by tenant, and the concept walk gates by tenant'
blocking:
  - '[ ] 19093 Delete the engine''s concept files and JSONL copy'
---
# The KB projects admitted concepts from the plane store

## Context

#471's option E′ makes the plane store the concept source. The default KB still reads concepts through an interim route over the engine pin. neomjs/neo#19096 settles two things for this leaf: the admission rule (OQ12, rule (a)) and the fresh-plane proof (G5).

## The Problem

- **The only KB concept source is a repository read.** The Engine profile routes `ConceptSource` over `resources/content/concepts`, wired in three places:
  - the route, `coreCorpusProfilePlan.mjs:56`;
  - the catalogue descriptor, `ExtractorCatalogue.mjs:282`;
  - the leaf, `sourcePaths.ConceptSource` at `knowledge-base/configBase.mjs:613`.

  `ConceptSource.mjs:40` also resolves that path from `projectRoot`. The mirror retirement (neomjs/neo#19159) deletes the path, and no other operator has one.
- **No KB source reads plane data.** No class under `ai/services/knowledge-base/source/` references `dataRoot` (@neo-opus-grace's `STEP_BACK`, point 2). The projection is a new source kind, not a route swap.
- **Nothing gates admission.**
  - `QueryService.mjs:193–200` boosts `type === 'concept'` without checking `validated`.
  - `resolveConcepts` (`conceptAnchoredRetrieval.mjs:109–168`) enumerates CONCEPT rows without that check.
  - `SearchService.mjs:563–571` returns the walk event even when the flat result is empty.

  Projecting the store as-is would put every discovered candidate (69+) into concept search and the walk.

## The Architectural Reality

- **Where the store lives.** The store is in the orchestrator's private plane root. The orchestrator already runs tenant embedding (`TenantRepoSyncService` → `VectorService`) and the graph projection (`ConceptIngestor`). Chroma is shared with kb-server.
- **Scoping.** `buildOwnedScopeFilter({tenantId, repoSlug})` scopes an embed run to its own tuple (`VectorService.mjs:174, :411, :498`).
- **The admission predicate the readers already apply:** `validated` is not `false` (`ConceptIngestor.mjs:131`, `GapInferenceEngine.mjs:311`). No row carries `validated: true`: the engine copy has 0 `true`, 6 `false` and 59 rows without the key.
- **Legacy vectors.** #419 AC-4 keeps the legacy `type: concept` vectors until this leaf's projection receipt. Legacy `kbSync` stays off (#411 AC-5, #419 AC-6).
- **neomjs/neo#19051's cutover predicate** requires that no runtime reader derives `resources/content` from `projectRoot`. `ConceptSource.mjs:40` is such a reader, and #459's census does not list it.

## The Fix

- **Projection.** An orchestrator-side concept projection reads the store through the store leaf's plane-member path and parses it with `ConceptIngestor`'s JSONL parse, so one parser feeds two projections. It embeds tenant-scoped `type: concept` vectors under `buildOwnedScopeFilter`.
- **Admission.** Rows whose `validated` is `false` are never projected into the KB. `resolveConcepts` skips them too, so a candidate that reaches the graph cannot reach `resolvedConcepts`.
- **Route retirement.** Removed: the Engine-pin route, its catalogue descriptor and the `sourcePaths.ConceptSource` leaf. `ConceptSource.mjs` goes too if nothing else routes to it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `type: concept` KB vectors | the plane store, via the store leaf's path | projected per tenant; admitted rows only | none: a seedless plane has none | ADR 0023 / 0024 (sibling leaf) | the projection receipt |
| `ask(conceptWalk: true)` / `resolveConcepts` (`conceptAnchoredRetrieval.mjs:109–168`) | this ticket | skips `validated: false` | — | — | AC-3's pair |
| The Engine-pin `ConceptSource` route and the `sourcePaths.ConceptSource` leaf (`coreCorpusProfilePlan.mjs:56`, `ExtractorCatalogue.mjs:282`, `knowledge-base/configBase.mjs:613`) | #282 / PR #423 | removed | — | — | the profile-plan spec and config parity |

Decision Record impact: aligned-with ADR 0019 (leaf removal and the use-site read); depends-on the ADR 0023 / 0024 amendment (sibling leaf).

## Acceptance Criteria

- [ ] **AC-1** The projection embeds tenant-scoped `type: concept` vectors from the store, using `ConceptIngestor`'s parse.
- [ ] **AC-2** A spec with two arms: a seeded row with no `validated` key is projected and admitted, and a discovered candidate (`validated: false`) is not.
- [ ] **AC-3** In `ask(conceptWalk: true)`, a candidate linked to an authorized file appears in none of `resolvedConcepts`, `conceptPath`, the references or the answer. An admitted seed does appear.
- [ ] **AC-4** The Engine profile no longer routes `ConceptSource`. The catalogue descriptor and the `sourcePaths.ConceptSource` leaf are removed, and no Brain reader resolves `resources/content/concepts`.
- [ ] **AC-5** *(deployed, fresh plane)* Use a plane bootstrapped without seeds, or `ai/examples/cloud-deployment/minimal-external-workspace`:
  - discovery appends a candidate to its store;
  - the KB holds no concept (the empty-KB control) while the tenant-scoped graph holds the candidate;
  - `query_documents type:concept` answers only once a concept is validated.
- [ ] **AC-6** *(deployed)* One orchestrator recreate, with no manual copy, leaves the store's node and edge counts unchanged.
- [ ] **AC-7** *(deployed, canonical plane)* The projection receipt counts concept vectors per tenant. #419 AC-4 retires the legacy `neo` concept vectors against it.

## Out of Scope

- The id scheme and the tenant gate (the store leaf, which blocks this one).
- The curation surface and promotion (OQ9).
- The engine's copies (neomjs/neo#19093).

unowned-rationale: no 13.2 dependency, because the mirror stays through the 13.2 cut (neomjs/neo#19051). A peer claims it at intake.

## Related

#471 (parent) · neomjs/neo#19096 (OQ1, OQ12, G5) · #282 · #419 · #411 · #425 · #459 · neomjs/neo#19093 · neomjs/neo#19159

Sweeps at 20:36Z: the #471 sweeps (20:33Z) cover this leaf. The live latest-open re-check found no concept-projection ticket. #459 is a sibling and does not list `ConceptSource`. Structure map: executed for #471, owner `ai/services/knowledge-base/source/` (sibling `ConceptSource.mjs`).

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("KB concept projection plane store admission validated not false ConceptSource route retirement conceptWalk candidate")`

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-24T20:37:10Z @neo-opus-vega added the `enhancement` label
- 2026-09-24T20:37:10Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:37:10Z @neo-opus-vega added the `architecture` label
- 2026-09-24T20:37:10Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:37:22Z @neo-opus-vega added parent issue #471
- 2026-09-24T20:37:24Z @neo-opus-vega marked this issue as being blocked by #473
- 2026-09-24T20:37:54Z @neo-opus-vega cross-referenced by #19093
- 2026-09-24T20:37:57Z @neo-opus-vega marked this issue as blocking #19093
- 2026-09-24T20:38:20Z @neo-opus-vega cross-referenced by #471

