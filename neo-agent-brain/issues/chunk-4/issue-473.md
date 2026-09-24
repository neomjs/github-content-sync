---
id: 473
title: 'Concept rows are keyed by tenant, and the concept walk gates by tenant'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-09-24T20:37:07Z'
updatedAt: '2026-09-24T20:37:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/473'
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
blockedBy: []
blocking:
  - '[ ] 474 The KB projects admitted concepts from the plane store'
---
# Concept rows are keyed by tenant, and the concept walk gates by tenant

## Context

Under #471's option E′, one plane store serves every tenant a deployment ingests. neomjs/neo#19096 G1 sets the contract: tenant-qualified rows, a re-key with a count receipt, and a two-tenant fixture that includes one principal in both tenants.

## The Problem

- **Ids are bare name slugs.** The 65 engine rows use ids like `neo-mjs`, and 182 edges reference them as `source` and `target`. The live store follows the same scheme at 136 nodes. So two tenants that declare the same name collide.
- **Visibility gates by user, not by tenant:**
  - `GraphService`'s enumeration (`GraphService.mjs:1165–1209`) and its node and edge visibility (`:1310–1341`) gate by the requester's user id.
  - `resolveConcepts` (`conceptAnchoredRetrieval.mjs:109–168`) clusters whatever `GraphService` returns.
  - The KB walk re-authorizes only the terminal file (`SearchService.mjs:519–550`).
  - So one principal with memberships in two tenants could walk from A's concept through B's. This is a prospective risk, not a measured leak (@neo-gpt, DC 18565631, DC 18586212).
- **The store path is derived, not declared.** `daemon.mjs:452` assigns `ConceptService.defaultConceptsDir = path.join(AiConfig.plane.dataRoot, 'concepts')`. The KB projection, a sibling leaf, would have to repeat that derivation.

## The Architectural Reality

- **Store:** `dataRoot/concepts/{nodes,edges}.jsonl` in the orchestrator's private plane root (`deploy/cloud/docker-compose.yml:495`). It is backed up as the `concepts` substrate (`restore.mjs:172`).
- **Writers:** `ConceptDiscoveryService` appends `validated: false` candidates (`:412`). Seeds carry no `validated` key.
- **Id readers:** `ConceptIngestor` (the graph projection), `GapInferenceEngine`, `conceptAnchoredRetrieval`, `conceptSliceBuilder` (the Sandman slice), and backup and restore.
- **ADR-0019 §5 and §10.5:** a plane-anchored path is one declared leaf with a `planeMember` decision, read at each use site.

## The Fix

- **Id scheme.** Ids are derived from `(tenantId, repoSlug?, name)`. A concept that spans repositories carries edges to several `repoSlug`s, not one owner. Rows gain provenance fields: tenant, origin (seed or discovery), and discovery source and time.
- **Re-key.** A one-shot migration re-keys nodes and edges together. It prints node and edge counts before and after, plus the dangling references, which must be 0. Every id reader listed above reads the new ids.
- **Tenant gate.** Concept enumeration, `resolveConcepts` and the walk gate by the requested tenant, not only by user id.
- **Declared path.** The store directory becomes one plane-member leaf, read at the use site by `ConceptService` and discovery, and later by the KB projection. The assignment in `daemon.mjs` goes.
- **Sink.** Discovery writes only to the store. Nothing writes concepts into a tenant repository (OQ2).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Concept row id (`nodes.jsonl` / `edges.jsonl`) | this ticket | tenant-qualified, with provenance fields | none; the migration re-keys | `ConceptOntology.md` via the ADR leaf | re-key receipt |
| Store directory (today `daemon.mjs:452`) | one AiConfig plane-member leaf (ADR-0019 §10.5) | read at each use site | none | ADR-0019 | the member-coherence boot check |
| Concept enumeration and walk (`GraphService.mjs:1165–1209`, `conceptAnchoredRetrieval.mjs:109–168`) | this ticket | gated by the requested tenant | refuse | — | the two-tenant fixture |

Decision Record impact: aligned-with ADR 0019; depends-on the ADR 0023 / 0024 amendment (sibling leaf).

## Acceptance Criteria

- [ ] **AC-1** Rows carry a tenant-qualified id and provenance fields, and the store's JSDoc documents the scheme.
- [ ] **AC-2** The fixture:
  - **Setup:** tenants A and B declare a concept with the same `name` and the same relative file path.
  - **Separation:** ids, aliases, edges, discovery dedupe, `query_documents` and `ask` results, and `conceptWalk` all stay separate.
  - **Queries:** querying A neither traverses B's private intermediate concept nor emits B's name in `resolvedConcepts` or `conceptPath`.
  - **Deletion:** deleting A leaves B intact.
- [ ] **AC-3** In the same fixture, one principal holds memberships in both A and B and queries each tenant in turn. A's ids, aliases, intermediate concepts, `resolvedConcepts`, `conceptPath` and delete result all exclude B, and the reverse also holds.
- [ ] **AC-4** The store directory is one plane-member leaf. `ConceptService` and discovery read it at the use site, and `daemon.mjs` no longer assigns `defaultConceptsDir`.
- [ ] **AC-5** A spec shows that discovery's sink resolves under that leaf, and that no concept write targets a repository path.
- [ ] **AC-6** *(deployed plane, post-merge)* The live store is re-keyed on the canonical plane. The receipt prints node and edge counts before and after, with 0 dangling references. A `concepts` backup then restores with the new ids.

## Out of Scope

- The KB projection and its admission rule (sibling leaf).
- Promoting candidates, and any curation surface (OQ9).
- The membership model itself (neomjs/neo#15605).

unowned-rationale: no 13.2 dependency, because the mirror stays through the 13.2 cut (neomjs/neo#19051). A peer claims it at intake.

## Related

#471 (parent) · neomjs/neo#19096 (G1, OQ2, OQ3) · #425 · neomjs/neo#15605

Sweeps at 20:36Z: the #471 sweeps (20:33Z) cover this leaf. The live latest-open re-check found no concept-id or tenant-gate ticket. Structure map: executed for #471. The owners are `ai/services/ConceptService.mjs`, `ai/services/ingestion/` and `ai/services/graph/`.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("concept rows tenant-qualified ids re-key two-tenant fixture same principal GraphService userId gate")`

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-24T20:37:09Z @neo-opus-vega added the `enhancement` label
- 2026-09-24T20:37:09Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:37:09Z @neo-opus-vega added the `architecture` label
- 2026-09-24T20:37:09Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:37:20Z @neo-opus-vega added parent issue #471
- 2026-09-24T20:37:24Z @neo-opus-vega marked this issue as blocking #474
- 2026-09-24T20:38:20Z @neo-opus-vega cross-referenced by #471

