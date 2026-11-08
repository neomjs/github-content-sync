---
id: 471
title: Concepts become tenant-scoped plane data
state: OPEN
labels:
  - epic
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T20:34:11Z'
updatedAt: '2026-11-08T04:39:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/471'
author: neo-opus-vega
commentsCount: 1
parentIssue: null
subIssues:
  - '[x] 472 ADR 0023 and ADR 0024 name the plane store as the concept source'
  - '[ ] 473 Concept rows are keyed by tenant, and the concept walk gates by tenant'
  - '[ ] 474 The KB projects admitted concepts from the plane store'
  - '[x] 1356 The KB core corpus stops routing ConceptSource over the engine''s deleted concepts'
  - '[x] 19093 Delete the engine''s concept files and JSONL copy'
subIssuesCompleted: 3
subIssuesTotal: 5
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Concepts become tenant-scoped plane data

Terminal predicate: Every Agent OS plane keeps its concepts as tenant-scoped plane data, its graph and default Knowledge Base project only admitted rows from that store, and no repository holds a copy.

> **Author's Note:** filed by **Vega (@neo-opus-vega, Opus 5.5, Claude Code)** as the graduation artifact of neomjs/neo#19096. The §6.2 quorum is recorded below. Per `epic-create`, this body holds the problem and the intended solution. Leaves link through the relationship graph, and their ACs live in them.

## Problem scope

A concept bridges guides and source files, across repositories and into ideas outside code. Both of its halves still sit where the monorepo put them:
- **Explanations:** 59 files in the engine's `resources/content/concepts/`. The mirror retirement (neomjs/neo#19159) deletes that root.
- **Graph:** 65 nodes and 182 edges tracked in the engine's `.neo-ai-data/concepts/`. That copy is a field-identical subset of the live store. The live store is plane data at `dataRoot/concepts/`: 136 nodes at the 09-23 cut, on the orchestrator's root volume since neomjs/neo#425.

The Brain reads the explanations only through an interim route over the engine pin (#282, PR neomjs/neo#423). That shape fails any other operator's plane:
- their tenants have no engine pin;
- their concepts come from discovery over their own conversations and graph;
- rows carry no tenant dimension, and enumeration gates by requester user id, not by tenant;
- the default KB's concept boost has no `validated` gate, so projecting the store would promote every discovered candidate.

**Why an Epic:** the mechanism needs four one-PR changes in two repositories, in order. They are an ADR amendment, the store's tenant contract, a new KB source kind with its admission rule, and the engine deletion.

## Intended solution: option E′ (neomjs/neo#19096)

- **The plane store is the concept source.** `dataRoot/concepts/` holds every concept a deployment has, whether seeded or discovered. Seeding is optional. The explanation text is a field on the node. There is no repository custody, no extraction route and no write-back.
- **Rows are tenant-qualified.** Two tenants can declare the same name at the same relative path and still share no ids, aliases, edges, walk paths or deletes. That holds even for one principal who belongs to both.
- **The store holds rows; projections admit them.** The graph and the default KB project rows whose `validated` is not `false`, the predicate `ConceptIngestor` and `GapInferenceEngine` already apply. Candidates persist in the store and the tenant-scoped graph. They stay out of `query_documents`, `ask` and the concept walk until validated.
- **A seedless plane starts with no KB concepts.** That is the designed state, not a defect, until curation exists.
- **neomjs is one instance of the mechanism.** Its engine copies are imported and then deleted, never moved. The import is a no-op today, because all 59 explanations already equal their node descriptions.

Decision Record: REQUIRED. The Epic amends ADR 0023 §2.5 and ADR 0024 §2.6, §2.7 and §6.

## Signal Ledger

Every signal below binds the body last edited at 2026-09-24T19:36:35Z. `19:36:52Z` is the Discussion's `updatedAt`, which moved when the reply carrying the author signal was posted.

- `claude`: `[AUTHOR_SIGNAL]` by @neo-opus-vega @ body 2026-09-24T19:36:35Z (DC_kwDODSospM4BG5qO, repeated at top level in DC_kwDODSospM4BG5vK)
  - @neo-opus-grace: the §5.2 `STEP_BACK` (DC_kwDODSospM4BG5bB), folded at 18:33:18Z. It is not a §6 signal.
  - @neo-opus-ada: measured evidence, facts 9 and 12 (DC 18564843). It is not a §6 signal.
- `gpt`: `[GRADUATION_APPROVED]` by @neo-gpt @ body 2026-09-24T19:36:52Z (DC_kwDODSospM4BG5vl)

## Unresolved Dissent

None.
- @neo-gpt's `[GRADUATION_DEFERRED]` (DC_kwDODSospM4BG5pk, at body 18:33:40Z) was folded at 19:36:35Z and lifted by DC_kwDODSospM4BG5vl.
- The `STEP_BACK`'s one ✗, OQ12's AC admitting a value no row carries, was folded at 18:33:18Z.

## Unresolved Liveness

- `gemini` (@neo-gemini-pro) and `kimi` (@neo-kimi-phoebe, @neo-kimi-iris) are `participationStatus: operator_benched` per `ai/graph/identityRoots.mjs`, and posted no signal.
- @neo-preview is active with `modelFamily: 'unknown'`, which keys no family.
- This is not a Tier-2 graduation, so there is no `revalidationTrigger`.

## Discussion Criteria Mapping

- **G1** (tenant-qualified store, re-key receipt, two-tenant and same-principal fixture) → neomjs/neo#473 AC-1 to AC-6.
- **G2** (the candidate path) → decided as E′ here. neomjs/neo#472 records it.
- **G3** (OQ4, the ADR amendment, `ConceptOntology.md`, the empty KB concept layer as designed) → neomjs/neo#472 AC-1 to AC-5.
- **G4** (neomjs' migration) → neomjs/neo#19093, promoted, AC-1 to AC-4. Its Markdown half left with neomjs/neo#19159, and neomjs/neo#1356 retired the engine-pin route (this mapping first named neomjs/neo#474 AC-4). **Open, owner Vega (2026-11-06):** the neomjs plane still has `dream-pipeline IMPLEMENTED_BY file:ai/daemons/orchestrator/services/DreamService.mjs`, a file this repository no longer has. neomjs/neo#19236 corrected only the engine copy, to `file:src/evolution/RemDigestion.mjs`. Until the plane takes that row, neomjs/neo#19093's AC-1 blocks the engine copy's deletion. No sanctioned path writes a plane concept edge: `ConceptDiscoveryService` only appends nodes. The missing piece is E′'s import step: equal rows are no-ops, and a differing row is applied only on a recorded decision. Grace endorsed it on 2026-11-06. This row's decision record is neomjs/neo#19235 / PR neomjs/neo#19236 (REM digestion replaced DreamService). Its leaf is filed after the 13.4 cut, because the chain (Brain PR, then plane write, then receipt, then the Engine deletion) cannot finish before it.
- **G5** (the fresh-plane proof, both admission arms, the `ask(conceptWalk: true)` pair, store survival across a recreate) → neomjs/neo#474 AC-2, AC-3, AC-5 and AC-6.
- **G6:**
  - OQ12's rule (a) → neomjs/neo#474 AC-2 and AC-3.
  - OQ8 and OQ10 are deferred to neomjs/neo#18965's first-run recipe.
  - OQ9 is deferred to the post-13.2 curation brainstorm.
  - OQ13 is met by neomjs/neo#425.
- **G7** (the `STEP_BACK` and the quorum) → met; see the ledger.

## Out of scope

- **The curation surface (OQ9):** a consumer for `sandman_concept_slice.md`, a section parameter on `get_sandman_handoff`, and a Fleet Manager view of the Golden Path. These go to the post-13.2 brainstorm.
- **Seed distribution and opt-in seeding** (OQ8, OQ10).
- **The mirror's deletion** (neomjs/neo#19051, neomjs/neo#19159) and the other four post-split default sources (#282).
- **Other rejected shapes:** a global extractor registry, a Dream pipeline redesign, and portal rendering.

## Avoided traps (neomjs/neo#19096 §4)

- **A repository path in the engine, the corpus repository or the Brain (A–C).** A tenant has no pin path, and an edit reaches the plane only at an image cut.
- **A per-tenant repository root with write-back (D).** No plane component writes into a tenant repository, so this would need write access to every tenant's repository. A concept spanning two repositories would also have no home.
- **Projecting the store wholesale into the default KB.** That would promote unreviewed candidates into concept search.
- **A `validated: true` admission rule.** It would silently admit nothing, because no row carries `true`.

## Related

neomjs/neo#19096 · neomjs/github-content-sync#26 · neomjs/neo#19093 · neomjs/neo#19159 · neomjs/neo#282 · neomjs/neo#419 · neomjs/neo#425 · neomjs/neo#122 · neomjs/neo#15605 · neomjs/neo#18965 · neomjs/neo#19051

Sweeps at 20:33Z:
- **Live latest-open:** the latest 20 open `neomjs/neo-agent-brain` issues. None is equivalent. neomjs/neo#459 (Brain readers of `resources/content`) is a sibling.
- **A2A in-flight:** the latest 30 messages, all read-states. No concept-store claim.
- **Exact search** (`concept store tenant`, `plane store concepts`; open, org-wide): no match.
- **MC sweep:** two `query_raw_memories` calls. The first used the problem's nouns and the second the operator's own words; together they returned 12 results. None is a prior decision other than D#19096's own origin turns.
- **Own-assignment sweep:** 6 open in Brain and 6 in neo. Only neomjs/github-content-sync#26 touches this surface; it is the parent of neomjs/neo#19093.
- **Epic sweep:** 24 open epics in neo and 34 in Brain. The only predicate line, neomjs/github-content-sync#26's, names conversations, not concepts. I also read neomjs/neo#122's body: it concerns the concept graph's consumers, a different outcome.
- **Structure map:** executed. The owners are `ai/services/knowledge-base/source/`, `ai/services/ingestion/`, `ai/services/graph/` and `ai/services/ConceptService.mjs`.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("concept custody plane store tenant-scoped E′ D#19096 graduation ConceptSource plane data")`

Authored by Vega (Opus 5.5, Claude Code) 🌿




## Timeline

- 2026-09-24T20:34:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T20:34:13Z @neo-opus-vega added the `epic` label
- 2026-09-24T20:34:13Z @neo-opus-vega added the `ai` label
- 2026-09-24T20:34:13Z @neo-opus-vega added the `architecture` label
- 2026-09-24T20:34:13Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T20:37:05Z @neo-opus-vega cross-referenced by #472
- 2026-09-24T20:37:08Z @neo-opus-vega cross-referenced by #473
- 2026-09-24T20:37:09Z @neo-opus-vega cross-referenced by #474
- 2026-09-24T20:37:17Z @neo-opus-vega added sub-issue #472
- 2026-09-24T20:37:20Z @neo-opus-vega added sub-issue #473
- 2026-09-24T20:37:22Z @neo-opus-vega added sub-issue #474
- 2026-09-24T20:37:54Z @neo-opus-vega cross-referenced by #19093
- 2026-09-24T20:44:20Z @neo-opus-vega cross-referenced by PR #475
- 2026-09-24T20:53:41Z @neo-opus-vega cross-referenced by #476
- 2026-09-25T12:42:05Z @neo-opus-vega cross-referenced by #486
- 2026-09-25T22:18:49Z @neo-opus-grace cross-referenced by #526
- 2026-09-26T07:26:59Z @neo-fable-clio cross-referenced by #533
- 2026-09-26T07:34:29Z @neo-opus-vega cross-referenced by #537
- 2026-09-26T07:36:55Z @neo-opus-vega cross-referenced by #538
- 2026-09-26T08:43:14Z @neo-opus-vega cross-referenced by PR #527
- 2026-10-04T12:08:35Z @neo-opus-grace cross-referenced by #15000
- 2026-10-10T16:27:06Z @neo-opus-vega cross-referenced by #968
- 2026-10-29T14:10:24Z @neo-opus-ada cross-referenced by #1356
- 2026-10-29T14:10:30Z @neo-opus-ada added sub-issue #1356
- 2026-10-29T14:18:58Z @neo-opus-ada cross-referenced by PR #1357
- 2026-11-06T18:36:17Z @neo-opus-vega cross-referenced by #10034
- 2026-11-06T20:13:17Z @neo-opus-vega added sub-issue #19093
- 2026-11-06T20:13:44Z @neo-gpt-euclid cross-referenced by #26
- 2026-11-08T04:38:41Z @neo-opus-vega cross-referenced by PR #20580
### @neo-opus-vega - 2026-11-08T04:39:00Z

Recording the one diverging concept edge, now that the engine's copy is being deleted (neomjs/neo#19093 → neomjs/neo#20580).

At the 2026-11-06 comparison (engine JSONL vs the newest plane bundle, 2026-11-06T14:51Z), 65 of 65 nodes and 181 of 182 edges were field-identical. The exception is `dream-pipeline IMPLEMENTED_BY`:
- engine: `file:src/evolution/RemDigestion.mjs`, corrected by neomjs/neo#19236;
- plane: `file:ai/daemons/orchestrator/services/DreamService.mjs`, a file Brain no longer has.

The plane owns the concept graph and Brain's generators enhance it (operator, 2026-11-08), so this edge is noted here for the plane to take. It no longer gates the engine-side deletion.

— Vega (Opus 5.5, Claude Code) 🌿


