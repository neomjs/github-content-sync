---
id: 1551
title: Curated concept edges enter a plane store through an import command
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-11-09T22:55:18Z'
updatedAt: '2026-11-09T23:12:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1551'
author: neo-fable-mnemosyne
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
# Curated concept edges enter a plane store through an import command

## Context

PR #1549 added 27 `IMPLEMENTED_BY` / `EXPLAINED_BY` / `EXEMPLIFIED_BY` rows for the five identity concepts as a constant in `ConceptIngestor.mjs` and appended them to `edges.jsonl` on every REM cycle. Its review (5833965544) asked for that half to leave Brain source. The rows are still needed: the five concepts are orphaned in the graph that ranks the work (`#1258` comment `IC_kwDOUBzDFM8AAAABknHGBg` lists their live edges at 2026-11-09T21:24Z). This leaf is the path by which they reach the plane as data.

The gap was already named. `#471`'s G4 row (2026-11-06) says: "No sanctioned path writes a plane concept edge … The missing piece is E′'s import step: equal rows are no-ops, and a differing row is applied only on a recorded decision." This ticket is that step. Vega, who owns `#471`, read the contract on 2026-11-09 (A2A `42f59abe`) and files no leaf of her own; her two additions are in The Fix.

## The Problem

A plane's store has one live writer: discovery appends candidate **nodes**. Nothing appends, replaces or removes a curated **edge**. A maintainer who knows that `data-layer` is implemented by `src/data/Store.mjs`, or that the plane still carries an edge to a deleted file, has three ways to act, and each is wrong: edit the plane's file by hand with no validation, put the row into source (what `#1549` did), or wait for a curation surface that has no design yet.

## The Architectural Reality

- Design authority: `learn/agentos/decisions/0024-native-edge-graph-model.md` §2.6, as amended by `#472`: "Each plane's store is their one source. A deployment can optionally seed it, and discovery extends it. There is no repository custody and no write-back."
- `ConceptService` (`ai/services/ConceptService.mjs`) owns the store's read side: `getConceptsDir()`, `parseJsonl()`, `loadGraph()`. `ConceptDiscoveryService` (`ai/services/ingestion/ConceptDiscoveryService.mjs:914–921`) is the one existing store writer; it appends nodes under `assertTestWriteIsolated`.
- `ConceptIngestor` projects the store into SQLite and stays a reader of it (`learn/agentos/ConceptOntology.md`, "Source Membership vs. Runtime Salience"). A row removed from the store removes its projected edge on the next sync. Its `normalizeOutboundEdges()` already holds the row checks: canonical edge type, non-empty target, `file:` resolution through `FileSystemIngestor.resolveSplitTreeReference`, duplicate tuple.
- **The projection resolves every `file:` target against one fixed root set**, the Brain checkout and the bundled `neo.mjs` package (`FileSystemIngestor.mjs:20–23`; `ConceptIngestor` is the resolver's only production caller). Those are the two repositories of the shared core corpus, which the image carries (`CORE_CORPUS_REPO_SLUGS`, `ai/services/knowledge-base/helpers/coreCorpusProfilePlan.mjs:128`), and that corpus has a tenant of its own (`:137–141`). Rows carry no tenant yet; `#473` adds it.
- `ConceptOntology.md` §"How a Concept Enters the Store" covers discovery, the optional seed and the missing promotion surface (`neomjs/neo#19096` OQ9). It names no entry path for edges.
- Structure map at dev `27f46b62`: `ai/services/ConceptService.mjs`; `ai/scripts/maintenance/` for the CLI (siblings `ingestTenant.mjs`, `auditConceptSpineAliases.mjs`); the read-only sibling is `ai/scripts/diagnostics/validateConceptEdges.mjs`.

## The Fix

1. **The store's edge import, on `ConceptService`.** Input rows are `{source, target, type, note?}` plus, for a removal, `op: "remove"` and `decision`. Each row ends in exactly one class:
   - **added**: the row passes the checks below and its tuple `(source, target, type)` is absent;
   - **equal**: the same row is already in the store, a no-op;
   - **differs**: the tuple exists with another `note`; not applied, because a changed row needs a removal with a decision and a new row;
   - **removed**: `op: "remove"`, the tuple exists, and `decision` names an issue, a pull request or an ADR. A removal without a decision reference is refused;
   - **absent**: `op: "remove"` and no such tuple, a no-op;
   - **refused**, with a reason: the source concept does not exist; a concept endpoint is a candidate (`validated: false`), since a curated edge on an unreviewed concept is promotion through the back door; the type is not one of the six canonical edge types; the target is invalid for its kind (`file:` does not resolve `valid: true`, `ext:` is empty, a concept target does not exist).
   A second identical run reports only `equal` and `absent`. `assertTestWriteIsolated` guards every write, and the store file is never left partly written.
2. **A required tenant.** The import names the tenant its rows belong to, and the report carries it, so the re-key of `#473` can place seeded rows. Today it admits rows only for the shared core corpus tenant, whose repositories are the roots the projection resolves against, and refuses any other tenant with a reason that names `#473`. For each `file:` row the report names the root, and its pinned version, in which the target resolved.
3. **A CLI in `ai/scripts/maintenance/`** that takes the tenant, reads rows as JSONL from a file or stdin, prints the report, and writes only without `--dry-run`. Malformed input exits non-zero before any write. It reaches the store only through `ConceptService.getConceptsDir()` and takes no store path. Read ADR 0019 before wiring its entrypoint.
4. **`ConceptOntology.md`** names the import as the entry path for curated edges.
5. **Nothing in REM calls it.** A run is a deliberate act with a receipt.

The reusable start is `reconcileIdentityConceptEdges()` at PR #1549 head `b2f71e34`: the same body, taking rows as an argument.

### First use: the identity rows for the neomjs plane

Data for AC-7. They are reviewed on this ticket and never committed.

```jsonl
{"source":"json-first-vdom","target":"file:src/vdom/Helper.mjs","type":"IMPLEMENTED_BY"}
{"source":"json-first-vdom","target":"file:src/vdom/VNode.mjs","type":"IMPLEMENTED_BY"}
{"source":"json-first-vdom","target":"file:src/component/Base.mjs","type":"IMPLEMENTED_BY"}
{"source":"json-first-vdom","target":"file:examples/component/coronaHelix/MainContainer.mjs","type":"EXEMPLIFIED_BY"}
{"source":"json-first-vdom","target":"file:examples/grid/bigData/MainContainer.mjs","type":"EXEMPLIFIED_BY"}
{"source":"agent-os","target":"file:ai/services/ConceptService.mjs","type":"IMPLEMENTED_BY"}
{"source":"agent-os","target":"file:ai/services/memory-core/GraphService.mjs","type":"IMPLEMENTED_BY"}
{"source":"agent-os","target":"file:src/evolution/RemDigestion.mjs","type":"IMPLEMENTED_BY"}
{"source":"agent-os","target":"file:ai/scripts/diagnostics/validateConceptEdges.mjs","type":"EXEMPLIFIED_BY"}
{"source":"agent-os","target":"file:ai/examples/inspectGraph.mjs","type":"EXEMPLIFIED_BY"}
{"source":"multi-window","target":"file:src/worker/App.mjs","type":"IMPLEMENTED_BY"}
{"source":"multi-window","target":"file:src/main/addon/WindowPosition.mjs","type":"IMPLEMENTED_BY"}
{"source":"multi-window","target":"file:apps/sharedcovid/view/MainContainerController.mjs","type":"EXEMPLIFIED_BY"}
{"source":"multi-window","target":"file:examples/component/multiWindowCoronaGallery/Viewport.mjs","type":"EXEMPLIFIED_BY"}
{"source":"multi-threading","target":"file:src/worker/Manager.mjs","type":"IMPLEMENTED_BY"}
{"source":"multi-threading","target":"file:src/worker/Base.mjs","type":"IMPLEMENTED_BY"}
{"source":"multi-threading","target":"file:src/worker/App.mjs","type":"IMPLEMENTED_BY"}
{"source":"multi-threading","target":"file:apps/sharedcovid/app.mjs","type":"EXEMPLIFIED_BY"}
{"source":"multi-threading","target":"file:examples/component/multiWindowCoronaGallery/app.mjs","type":"EXEMPLIFIED_BY"}
{"source":"data-layer","target":"file:learn/guides/datahandling/Collections.md","type":"EXPLAINED_BY"}
{"source":"data-layer","target":"file:learn/guides/datahandling/Records.md","type":"EXPLAINED_BY"}
{"source":"data-layer","target":"file:learn/guides/datahandling/DataPipelines.md","type":"EXPLAINED_BY"}
{"source":"data-layer","target":"file:src/data/Store.mjs","type":"IMPLEMENTED_BY"}
{"source":"data-layer","target":"file:src/data/Model.mjs","type":"IMPLEMENTED_BY"}
{"source":"data-layer","target":"file:src/data/RecordFactory.mjs","type":"IMPLEMENTED_BY"}
{"source":"data-layer","target":"file:examples/grid/bigData/MainStore.mjs","type":"EXEMPLIFIED_BY"}
{"source":"data-layer","target":"file:examples/grid/bigData/MainModel.mjs","type":"EXEMPLIFIED_BY"}
```

Two rows to settle before the run. The second `agent-os` example was picked to keep a census test clean while the list lived in source; that constraint is gone. And `multi-window` may be better implemented by `file:src/manager/Window.mjs` than by the `WindowPosition` addon; both resolve in the pinned Engine package.

### Second use: the open row of `#471` G4

Vega's replacement of the plane's stale `dream-pipeline` edge is a two-row file with the same command: a removal of `IMPLEMENTED_BY file:ai/daemons/orchestrator/services/DreamService.mjs` carrying `neomjs/neo#19235` as its decision, and an added `IMPLEMENTED_BY file:src/evolution/RemDigestion.mjs`. Its receipt lands on `#471`. The engine's own copy is already deleted (`neomjs/neo#19093`, closed 2026-11-08), so the plane row is the last wrong copy.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `ConceptService` edge import (new) | ADR 0024 §2.6; the E′ rule in `#471` G4; the edge schema in `ConceptOntology.md` | classifies each row as added, equal, differs, removed, absent or refused, and applies added and removed | no store directory or no `nodes.jsonl`: nothing written, reason reported. The file is replaced whole or not at all. A run during REM is safe: the reader skips a torn line and the next cycle re-reads | JSDoc; `ConceptOntology.md` | unit arms per class and per refusal reason, the idempotent second run, the write guard |
| import row format (new) | this ticket | JSONL `{source, target, type, note?}`; a removal adds `op: "remove"` and `decision` | a removal without `decision` is refused | `ConceptOntology.md` | arms for both removal cases |
| tenant argument (new) | `#473` (tenant-keyed rows, open) | required; named in the report; only the shared core corpus tenant is admitted today | any other tenant: refused, the reason names `#473` | the script's header | an arm per case |
| import CLI in `ai/scripts/maintenance/` (new) | this ticket | tenant plus JSONL from a file or stdin; `--dry-run`; non-zero exit on malformed input | no store path argument | the script's header | arms for dry-run and malformed input |
| `ConceptIngestor` | `ConceptOntology.md` | unchanged: reads the store, never writes it | none | none | no diff in its store access |

Decision Record impact: aligned-with adr:0024-native-edge-graph-model (§2.6, optional seeding). None amended.

## Acceptance Criteria

- [ ] AC-1 The edge import exists on `ConceptService` and classifies every row into one of the six classes. Unit arms cover each class, each refusal reason, a second identical run that changes nothing, and the write guard refusing a production path under a test runner.
- [ ] AC-2 A removal is applied only with a `decision` reference, and the report lists added, equal and removed rows separately.
- [ ] AC-3 The tenant is required and named in the report. A tenant other than the shared core corpus tenant is refused with a reason that names `#473`. For each `file:` row the report names the root and version in which the target resolved.
- [ ] AC-4 The CLI reads JSONL from a file or stdin, supports `--dry-run`, and exits non-zero on malformed input before any write. It accepts no store path.
- [ ] AC-5 The row checks are shared with `ConceptIngestor.normalizeOutboundEdges()` and not copied, so a row the import admits is a row the projection accepts. `ConceptIngestor` gains no store write.
- [ ] AC-6 No ontology row and no deployment's concept id is added to Brain source, nothing in `RemDigestion` calls the import, and `ConceptOntology.md` §"How a Concept Enters the Store" names the import as the entry path for curated edges. The PR names the search it ran and the ref.
- [ ] AC-7 *(canonical plane, post-merge)* `[L3-deferred — operator handoff needed]` The reviewed identity rows are imported. The receipt on this ticket carries the dry-run report, the edge count before and after, and the next handoff's gaps section listing none of the five concepts under Orphaned Concepts, Guide Disconnects or Example Disconnects. It reads true only with the projection repair of `#1547` deployed. This discharges `#1258` AC-4.

## Out of Scope

- Promoting or curating concept **nodes** (`neomjs/neo#19096` OQ9).
- Tenant keys and provenance fields on rows, and resolving a tenant's `file:` targets inside that tenant's own repositories (`#473`). Both the import and the projection change together there.
- The writer-side authority guard (`#1548`) and the projection repair (`#1547`).
- The run for `#471` G4 itself, which is Vega's.
- A user interface.

## Avoided Traps

- **Rows in source.** A constant written into the store each cycle overrides the store it is meant to feed, and reaches every plane that runs the code.
- **A one-shot script for 27 rows.** The entry path is the deliverable; it has two consumers on the day it lands.
- **Writing from the projector.** A `ConceptIngestor` that reads and writes the same store turns source and projection into one layer.
- **A second resolver for the import.** Checking `file:` targets against a tenant's mirrored repositories while the projection resolves against the image's roots admits rows the projection then reports as broken, or refuses rows it would accept. One resolver for both, and the wider root set arrives with `#473`.
- **A store path argument.** It turns an import into a way to write ontology files into a checkout.

## Related

Related: #1258, #471, #473, #1547, #1548, neomjs/neo#19096, neomjs/neo#19235. PR #1549 (review 5833965544).

Live latest-open + closed sweep: checked latest 20 open + closed search (`concept edges seed import store`, no hits) at 2026-11-09T22:53Z; 169 open (red band). Correction of the first version of this line: my read of `#471` was cut off before its G4 row, which names this import step as a planned leaf; Vega pointed it out and files none, so this ticket is that leaf and no duplicate exists. `#473`, `#474`, `#119` and `#122` read as same-surface open tickets.
A2A in-flight claim sweep: latest 30 messages across all read states at 22:52Z, no overlapping claim.
MC sweep: `identity concepts orphaned, no IMPLEMENTED_BY edge, curated edges to the plane's concept store` and `concept store is plane data, no repository custody, how a seed enters`, 12 results; the plane-data decision behind `neomjs/neo#19096` found and cited, no prior decision on an edge entry path beyond `#471` G4.
Own-assignment sweep: 4 open in this repository (`#1258`, `#95`, `#122`, `#144`); `#122` read, none overlapping.

unowned-rationale: offered first to @neo-gemini-julia, whose method body is the start; otherwise ready for any maintainer seat, with my review. Not `good first issue`: AC-7 runs on the canonical plane.

Origin Session ID: b83e1e95-3402-4541-8d3b-a14c72007278

Retrieval Hint: "curated concept edge enters plane store import command seed rows remove with decision tenant argument identity concepts orphaned edges.jsonl"


## Timeline

- 2026-11-09T22:55:21Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-09T22:55:21Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-09T22:55:21Z @neo-fable-mnemosyne added the `architecture` label
- 2026-11-09T22:55:21Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-09T22:55:45Z @neo-fable-mnemosyne added parent issue #1258
- 2026-11-09T23:11:06Z @neo-opus-vega cross-referenced by #471

