---
id: 472
title: ADR 0023 and ADR 0024 name the plane store as the concept source
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T20:37:04Z'
updatedAt: '2026-09-25T11:22:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/472'
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
  - '[ ] 19093 Delete the engine''s concept files and JSONL copy'
closedAt: '2026-09-25T11:22:20Z'
---
# ADR 0023 and ADR 0024 name the plane store as the concept source

## Context

neomjs/neo#19096 graduated option E′ into #471: concepts are tenant-scoped plane data. The Discussion recorded `Decision Record: REQUIRED`. Two accepted ADRs and the ontology guide still describe the opposite target.

## The Problem

A fresh agent reading these would rebuild the repository shape E′ rejected:
- **ADR 0023 §2.5.** The knowledge-graph row targets *"content-SSOT de-duplicated"*, meaning the Markdown explanations as the source. Its V-B-A note says *"The content-should-be-SSOT remains a target."*
- **ADR 0024 §2.6** calls the ontology *"git-versioned, PR-reviewable source membership at `.neo-ai-data/concepts/`"*.
- **ADR 0024 §2.7** lists a *Version-controlled ontology* provenance path.
- **ADR 0024 §6** leaves the content-as-SSOT unification open.
- **`learn/agentos/ConceptOntology.md`** L3, L89 and L157 describe a version-controlled ontology read from `.neo-ai-data/concepts/*.jsonl`.

## The Architectural Reality

- The live store is `dataRoot/concepts/`, on the orchestrator's root volume since #425. The engine copy is a field-identical subset of it (neomjs/neo#19096 fact 9).
- The source→projection rule itself stands in both ADRs. Only the source changes: the store is the source, and the graph and KB are its projections.

## The Fix

These are amendments, not supersessions:
- **ADR 0023 §2.5.** The knowledge-graph target becomes three things: the tenant-scoped plane store as the source, the explanation as a node field, and projections that admit rows whose `validated` is not `false`. The V-B-A note is replaced to match.
- **ADR 0024 §2.6 and §2.7.** The storage row names the store. The provenance path becomes an optional seed.
- **ADR 0024 §6.** The open content-as-SSOT item is recorded as decided.
- **`ConceptOntology.md`.** Lines L3, L89 and L157 are rewritten.
- **The seedless plane.** Both ADRs say a seedless plane has no KB concepts by design until a curation surface exists. That surface is neomjs/neo#19096 OQ9, a post-13.2 brainstorm.
- **Provenance.** Each amendment carries a dated line citing neomjs/neo#19096 and this ticket.

Decision Record impact: amends ADR 0023, amends ADR 0024.
Decision Record: Required: this ticket (the amendment is the record).

## Acceptance Criteria

- [ ] **AC-1** ADR 0023 §2.5's knowledge-graph row and its V-B-A note name the tenant-scoped plane store as the concept source and the explanation as a node field.
- [ ] **AC-2** ADR 0024 §2.6, §2.7 and §6 say the same, and §6 no longer lists content-as-SSOT as open.
- [ ] **AC-3** `ConceptOntology.md` no longer describes a version-controlled ontology.
- [ ] **AC-4** Both ADRs state the admission predicate (`validated` is not `false`), and state that a seedless plane has no KB concepts by design.
- [ ] **AC-5** Each amendment carries a dated line citing neomjs/neo#19096 and this ticket. It merges before or together with neomjs/neo#19093.

## Out of Scope

- Code: the store contract, the KB projection and the engine deletion are sibling leaves under #471.
- The curation surface (OQ9).

## Related

#471 (parent) · neomjs/neo#19096 (G2, G3, OQ4) · neomjs/neo#19093 · #425

Sweeps at 20:36Z: the #471 sweeps (20:33Z) cover this leaf. The live latest-open re-check found no ADR 0023 / 0024 amendment ticket. Structure map: N/A, docs only.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("ADR 0023 ADR 0024 amendment plane store concept source content-SSOT E′")`

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-24T20:37:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T20:37:06Z @neo-opus-vega added the `documentation` label
- 2026-09-24T20:37:06Z @neo-opus-vega added the `enhancement` label
- 2026-09-24T20:37:06Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:37:06Z @neo-opus-vega added the `architecture` label
- 2026-09-24T20:37:06Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:37:17Z @neo-opus-vega added parent issue #471
- 2026-09-24T20:37:54Z @neo-opus-vega cross-referenced by #19093
- 2026-09-24T20:37:55Z @neo-opus-vega marked this issue as blocking #19093
- 2026-09-24T20:38:20Z @neo-opus-vega cross-referenced by #471
- 2026-09-24T20:44:20Z @neo-opus-vega cross-referenced by PR #475
- 2026-09-25T10:56:38Z @neo-opus-vega referenced in commit `7a8416b` - "docs(adr): the amendments state #473 and #474 as open targets, not present fact (#472)

Clio's RA-2 on PR #475: the graph projection applies the validated predicate today (ConceptIngestor); the KB projection is #474's target, and tenant keys are #473's. One clause each, no other text."
- 2026-09-25T11:22:20Z @tobiu referenced in commit `77bfa56` - "Merge pull request #475 from neomjs/vega/472-plane-store-concept-source

docs(adr): ADR 0023 and ADR 0024 name the plane store as the concept source (#472)"
- 2026-09-25T11:22:20Z @tobiu closed this issue

