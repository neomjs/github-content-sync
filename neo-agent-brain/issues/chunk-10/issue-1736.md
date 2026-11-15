---
id: 1736
title: 'A code index of one Neo tree: class card, callers, subsystem scene'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-15T01:14:24Z'
updatedAt: '2026-11-15T01:31:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1736'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: null
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
# A code index of one Neo tree: class card, callers, subsystem scene

## Context

Discussion neomjs/neo#20581 designs a code graph for Neo. Its return for peers is unmeasured, and its comparison (leaf 0) cannot run a "structural scenes" arm before an extractor exists. The operator's input of Nov 15, recorded there: the graph is first an enabler for this team (his examples: reading the architecture of the dock layouts or of the Gantt family), a side quest of a couple of days, worth it at a high return for us alone.

This ticket is that first slice: leaf 0's instrument. It adds no graph store, no node type and no loaded rule, so the Discussion's open questions stay open.

`unowned-rationale: for maintainer self-selection. Julia holds the parser seam this slice is built on (Discussion comment 19622666). No good first issue: the slice sits under a sandbox that has not graduated, and its follow-up run needs a seat-side comparison.`

## The Problem

A peer who has to understand a subsystem greps and reads many files. The Knowledge Base returns chunks by similarity, and `get_class_hierarchy` returns inheritance. Nothing answers who calls a method, which writes trigger a config hook, or what a folder's classes look like as one picture.

Generic code-graph tools do not carry Neo's idioms. Measured on engine `src/` at `53a9533069` against an acorn walk of the same 545 files (table in Discussion comment 19621464):

| | acorn walk | Graphify 0.9.109 | codegraph 1.6.2 |
|---|---|---|---|
| Classes that extend another | 484 | 0 inheritance edges | 0 edges |
| Method → callee pairs through `this` | 2,104 | 1,597 bound | 2,072 bound |
| through `me` | 3,853 | 4 bound | 2,124 bound |
| through `super` | 841 | 1 bound | 21 bound |
| Hook methods (`afterSet*`, `beforeSet*`, `beforeGet*`) | 1,110 | 5 with a caller | 17 with a caller |

A hook has no textual caller: the config system calls it when its config is written. An index that does not know this reports 1,100 dead methods.

## The Architectural Reality

Verified on Brain `dev` at `2cb8e911`.

- `ai/services/knowledge-base/parser/SourceParser.mjs:20-144`: `inspectSourceModule(content, filePath, {strict})` is module-private. It parses one file with acorn and returns `className`, `superClassReference`, `imports`, `configNode`, `methodNodes` and `propertyNodes`. Its JSDoc says why it is the one class-description universe: a separate hierarchy scan once produced different class totals from the chunks.
- The same file imports `neo.mjs/src/core/Base.mjs` and the Knowledge Base logger and exports a `Neo.setupClass` singleton (`:2-3`, `:342`), so nothing in it runs without booting Neo.
- `ai/services/knowledge-base/helpers/repositoryClassHierarchyResolver.mjs` resolves an import-bound superclass across a repository (`resolveRelativeModule` `:60`, `resolveSuperClass` `:89`) and reaches the parser through `SourceParser.describeClass()` (`:197`). Its id and version are materialization input (`:5-13`).
- `ai/services/knowledge-base/helpers/chunkRowMetadata.mjs:4-12` is the precedent for the lift: pure, Neo-free logic moved out of a singleton so a caller and a spec can import the producer directly.
- `ai/scripts/diagnostics/` holds read-only tools of the same kind (`structureMap.mjs`, `consumerRelevanceCensus.mjs`).

## The Fix

Julia's seam from Discussion comment 19622666, in three parts.

1. **Lift.** Move `inspectSourceModule` into `ai/services/knowledge-base/helpers/inspectSourceModule.mjs`, importing acorn only. Signature: `inspectSourceModule(content, filePath, {strict = false, extractCalls = false})`. A parse failure is returned or thrown as a coded result; `SourceParser` keeps the logging. `SourceParser` imports the helper and passes no `extractCalls`, so its chunks do not change.
2. **Extract.** With `extractCalls: true` the same pass also reads, per class: `mixins` and `ntype` from the config block; each config with its reactive marker (`x_`); per method the calls through `this`, through `super`, and through `me` where the method binds `me` to `this`; and the config writes (`this.x = …`, `me.x = …`, `set({x: …})`).
3. **Index and read.** `ai/services/knowledge-base/helpers/sourceTreeIndex.mjs` exports `indexSourceTree({rootDir})`: it walks a directory on disk, runs the extraction per file and resolves across the tree. `extends` goes through the existing resolver's functions, never a third resolution. Calls are resolved along the class chain including mixins. A hook is bound to its config when the chain declares that config. What cannot be resolved stays a row with its text and a reason; nothing is guessed. Three reads sit on the index and are exposed by `ai/scripts/diagnostics/codeIndex.mjs` (npm script `ai:code-index`):
   - `card <ClassName>`: file, chain, mixins, configs with their hooks, methods with `overrides` and `callsSuper`.
   - `callers <ClassName#method>`: call sites with file and line; for a hook, the writes of its config, labelled as possible dispatch. A write does not prove the hook ran: `beforeSet` can cancel it, and `afterSet` runs only when the value changes (`src/Neo.mjs`, the generated setter).
   - `scene <folder>`: the folder's classes, their inheritance and mixin edges inside and across the folder border, and the methods called from outside it.

Pre-Flight (structural fast-path): `helpers/inspectSourceModule.mjs` and `helpers/sourceTreeIndex.mjs` match the sibling pattern of `helpers/chunkRowMetadata.mjs` and `helpers/repositoryClassHierarchyResolver.mjs`; `scripts/diagnostics/codeIndex.mjs` matches `scripts/diagnostics/structureMap.mjs`. No novel directory.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `inspectSourceModule` (new pure export) | `SourceParser.mjs:5-20` JSDoc | same return shape as today; `extractCalls: true` adds `calls`, `configWrites`, `mixins`, `ntype` | unparsable file: coded result, no throw unless `strict` | JSDoc | existing parser and resolver specs pass unchanged |
| `indexSourceTree({rootDir})` (new) | this ticket | a plain object: classes, methods, calls, hooks, unresolved rows | missing directory: coded error | JSDoc | unit spec on a fixture tree |
| `npm run ai:code-index -- --root <dir> <card\|callers\|scene> <arg> [--json] [--log <file>]` (new) | this ticket | text by default, JSON with `--json`; a cut result says how many rows it left out | unknown symbol: the candidates by name, exit code 1 | script header | run on engine `src/` |

## Decision Record impact

none. The slice writes to no store. The ADR amendment for code facts in the graph is a leaf of the Discussion and is not started here.

## Decision Record

Not needed for this slice. The Discussion's own target (an epic whose first leaf amends the graph's decision record) is unchanged and waits for its graduation.

## Acceptance Criteria

- [ ] AC-1: `inspectSourceModule` is exported from a module that imports acorn only. `SourceParser.spec.mjs` and `repositoryClassHierarchyResolver.spec.mjs` pass without a change to their assertions.
- [ ] AC-2: on a fixture tree, `indexSourceTree` resolves a superclass imported under another name, a mixin, a call through `me` bound to `this`, a `super` call, and binds an `afterSet` hook to its reactive config. A call it cannot resolve is a row with a reason.
- [ ] AC-3: `callers` on a hook lists the writes of its config with file and line, each labelled as possible dispatch and never as a call, on the fixture and on one named engine class.
- [ ] AC-4: on engine `src/`, the script prints for the indexed revision: classes with a resolved superclass, of 484 at `53a9533069`; bound call pairs per receiver, of the walk's 2,104, 3,853 and 841; hooks bound to a config, of 1,110. The PR body carries these numbers for its own revision. There is no threshold: the numbers are the reading.
- [ ] AC-5: the script runs without the plane, without network and without booting Neo, and indexes engine `src/` in under two seconds on a seat machine.
- [ ] AC-6: with `--log <file>` each read appends one JSON line: the read, its argument, the origin, commit and extractor version, the result's size in bytes, its unresolved targets, whether it was cut, and its duration. The time to build the index is logged apart from the time of each read.
- [ ] AC-7 (post-merge, not a close condition): the Discussion's pilot runs on the slice in Sophie's design (Discussion comment 19622732): one subsystem orientation and one historical change-impact question with a hidden accepted fix and an expected-evidence list frozen before either arm; fresh sessions of the same model and effort; one receipt per run. The design is Sophie's; the run is the Discussion author's to do or to route.

## Out of Scope

- Any store: the Native Edge Graph, Chroma metadata, a file kept between runs.
- An MCP tool. The reads stay a script until the comparison has run.
- Events, `ntype` and `module` references as edges, near-duplicate bodies, links to pull requests and tickets: later leaves of the Discussion.
- Changing what the hierarchy resolver resolves, or its id and version.

## Avoided Traps

- **A second AST walk beside the parser.** It drifted once already (`SourceParser.mjs:9-11`); the lift keeps one universe.
- **Adopting a generic tool's extraction.** Both were measured on this tree (the table above) and neither sees inheritance or hooks.
- **A threshold on the bound share.** The share is what the comparison reads; a gate would decide it before it is measured.
- **Name-only binding.** Ten classes define `applyRecordConfigs`; a call binds through the receiver's class chain or stays unresolved.

## Related

neomjs/neo#20581 (the Discussion; comments 19621464, 19622666, 19622732, 19527894)

Live latest-open + closed sweep: checked the latest 20 open issues in `neomjs/neo-agent-brain` (#1548 to #1733) and a closed search (`code index callers class hierarchy acorn call graph`) plus an org-wide search at 2026-11-15T01:25Z; no equivalent and no prior exclusion found. Backlog: 156 open, red band; no owning ticket or epic exists for a code index, so this one is filed.
A2A in-flight claim sweep: latest 30 messages, all read-states, at 01:25Z; no claim or intent on a code index.
MC sweep: `query_raw_memories` on "peers grep and read many files to find callers of a method, hooks and overrides", 5 results; only the sessions that produced the Discussion, no prior decision.
Own-assignment sweep: 5 open in `neomjs/neo-agent-brain` (#95, #122, #144, #1258, #1562), none on this surface.
Structure map: `npm run ai:structure-map -- --files --loc` at `2cb8e911`; owning folders `ai/services/knowledge-base/helpers` and `ai/scripts/diagnostics`.

Origin Session ID: 0bb3dc6b-4cd8-4925-8430-0e58a7ecf26b

Retrieval Hint: "code index class card callers subsystem scene inspectSourceModule lift", "leaf 0 instrument Discussion 20581"

## Timeline

- 2026-11-15T01:14:26Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-15T01:14:26Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-15T01:14:26Z @neo-fable-mnemosyne added the `agent-os` label

