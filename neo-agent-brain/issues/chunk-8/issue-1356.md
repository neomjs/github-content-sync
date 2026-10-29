---
id: 1356
title: The KB core corpus stops routing ConceptSource over the engine's deleted concepts
state: CLOSED
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T14:10:22Z'
updatedAt: '2026-10-29T18:33:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1356'
author: neo-opus-ada
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
  - '[ ] 1346 GuideToolParity reads NeuralLink.md from a current installed engine'
closedAt: '2026-10-29T18:33:25Z'
---
# The KB core corpus stops routing ConceptSource over the engine's deleted concepts

## Context

This is the route-retirement item of #474, split out with its author's agreement (@neo-opus-vega, A2A 2026-10-29T14:08Z). It blocks #1346, the Brain's engine pin move.

neomjs/neo#19322 deleted the engine's `resources/content` on 09-29, including the 59 concept pages. The plane store holds them (neomjs/neo#19159's note on neomjs/neo#19093). Their replacement read is #474's projection.

## The Problem

`createKnowledgeBase` compiles the engine corpus at the Brain's `neo.mjs` pin. Its Engine profile still routes `ConceptSource` over `resources/content/concepts`. At any engine past 09-29 that route throws: `imageRevisionReader` `lstat` → `ENOENT`, measured at `cc24c59e0d` in `DatabaseService.sync.spec` › "the exact Engine pin and Brain revision compile separately without consulting SourceRegistry". So the pin cannot move, and the KB keeps compiling a 09-25 engine.

## The Architectural Reality

Census of the route at `dev@2ed12394`:

| surface | site |
|---|---|
| the route and its scan root | `helpers/coreCorpusProfilePlan.mjs:97` (`route('ConceptSource', …)`) and `:88` (`roots`) |
| the catalogue descriptor | `source/ExtractorCatalogue.mjs:285–293`, named in its `:259` docblock |
| the class | `source/ConceptSource.mjs`, which resolves the path from `projectRoot` at `:40` (a neomjs/neo#19051 cutover reader) |
| the export | `source/_export.mjs` (import, both export lists, docblock) |
| the config default | `sourcePaths.ConceptSource` (`ai/mcp/server/knowledge-base/configBase.mjs:613`) |
| a KB-relevance prefix | `PrimaryRepoSyncService.mjs:49` (`'resources/content/concepts/'`) |
| docs | `parser/identity-tuple.md:26` (the default-source list) |
| tests | `source/ConceptSource.spec.mjs` (+ its `shared-config-write-ceilings.json` entry), `extractorCatalogue.spec`, `SourcePathsConfig.spec`, `SourceRegistry.spec`, `coreCorpusSources.spec` (Engine-profile concept fixture), `PrimaryRepoSyncService.spec` |

`embedKnowledgeBase` is additive ("routine stale deletion remains disabled"). Retiring the route therefore removes no vector: the legacy `type: concept` rows stay, as #419 AC-4 requires, until #474's projection receipt.

## The Fix

Remove every row of the census. `ConceptSource.mjs` and its spec are deleted, not left orphaned, so no Brain code derives `resources/content/concepts` from `projectRoot`. The KB-relevance prefix goes too: no KB source reads that root any more.

Kept on purpose:
- dated records: `MigrationPath.md:28` (a fact about an older pin) and ADR 0023's V-B-A notes;
- the legacy concept-vector control in `retireLegacyConversationRows.spec`, which #419 AC-4 keeps;
- negative pins that assert nothing routes the path.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Evidence |
|---|---|---|---|---|
| Engine-profile `ConceptSource` route and root | #474 (parent row), split here | removed | none: no engine past 09-29 has the directory | `coreCorpusSources.spec` negative pin; `DatabaseService.sync.spec` at the new pin, via #1346 |
| `ExtractorCatalogue` `ConceptSource` descriptor and `ConceptSource.mjs` | #474 (parent row) | deleted | — | `extractorCatalogue.spec`, `SourceRegistry.spec` |
| `sourcePaths.ConceptSource` default | #474 (parent row); ADR 0019 (declarative leaf, use-site reads) | removed from the leaf's default object | an overlay still naming it is an inert key | `SourcePathsConfig.spec`, `ai:lint-config-template-ssot` |
| legacy `type: concept` vectors | #419 AC-4 | unchanged | — | `embedKnowledgeBase`'s additive contract |

Decision Record impact: aligned-with ADR 0019 (leaf default change, no antipattern), aligned-with `adr:agentos-extraction-topology`.

## Acceptance Criteria

- [ ] AC-1: no Brain source routes, declares or reads `resources/content/concepts`. A `git grep` finds only the dated records, the legacy-vector control and the negative pins named under The Fix.
- [ ] AC-2: `ConceptSource.mjs`, its spec and its ceiling entry are deleted, and the catalogue, export, registry and source-path specs pass without it.
- [ ] AC-3: the Engine profile's corpus test keeps its ADR and test semantics, and asserts that no concept page is routed.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds.
- [ ] AC-5 (via #1346): `DatabaseService.sync.spec` passes at engine `cc24c59e0d`.

## Out of Scope

- #474's projection, admission and deployed ACs.
- Deleting legacy concept vectors (#419 AC-4).

## Related

Parent epic: #471. Split from: #474. Unblocks: #1346. Engine side: neomjs/neo#19322, neomjs/neo#19159, neomjs/neo#19051.

Live latest-open + closed sweep: latest open Brain issues + a closed/open search for `ConceptSource` at 2026-10-29T14:10Z. The hits (#474, #471, #402) are the parent and its epic, with no equivalent. A2A: @neo-opus-vega's agreement (14:08Z). Own assignments: #1346 (blocked by this). Structure map: N/A, since this deletes files and adds or moves none.

Retrieval Hint: "ConceptSource route retirement resources/content/concepts engine pin move KB core corpus ENOENT split from #474"


## Timeline

- 2026-10-29T14:10:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T14:10:24Z @neo-opus-ada added the `bug` label
- 2026-10-29T14:10:24Z @neo-opus-ada added the `ai` label
- 2026-10-29T14:10:25Z @neo-opus-ada added the `architecture` label
- 2026-10-29T14:10:30Z @neo-opus-ada added parent issue #471
- 2026-10-29T14:10:33Z @neo-opus-ada marked this issue as blocking #1346
- 2026-10-29T14:17:06Z @neo-opus-ada cross-referenced by #1346
- 2026-10-29T14:18:58Z @neo-opus-ada cross-referenced by PR #1357
- 2026-10-29T16:10:33Z @neo-opus-ada cross-referenced by #201
- 2026-10-29T16:12:50Z @neo-opus-ada cross-referenced by #1360
- 2026-10-29T16:28:51Z @neo-opus-ada referenced in commit `07b2a5e` - "Merge dev into ada/1356-retire-concept-route: pick up the bootstrap-flag leak fix (#1356)

dev carries #1354 (Resolves #1353), whose absence turned this PR's unit
comparison red: the backup empty arm read source-collection-unresolved
after ChromaManager.spec's create-path arm leaked the flag."
- 2026-10-29T18:33:25Z @tobiu referenced in commit `838f244` - "fix(knowledge-base): the core corpus stops routing ConceptSource over the engine's deleted concepts (#1356) (#1357)

The Engine profile routed ConceptSource over resources/content/concepts.
neomjs/neo#19322 deleted that tree on 09-29 (the plane store holds the
concept pages), so createKnowledgeBase throws ENOENT at any engine past
the Brain's 09-25 pin and the pin cannot move (#1346).

Removed with it: the profile root, the ExtractorCatalogue descriptor,
ConceptSource.mjs and its spec (deleted, not orphaned, so no Brain
reader derives the path from projectRoot), the sourcePaths.ConceptSource
default, the export entries, the KB-relevance prefix in
PrimaryRepoSyncService and the spec rows that enumerated the source.
The Engine-profile corpus test keeps a concept page in its fixture and
now asserts that nothing routes it. Legacy type: concept vectors stay:
embedKnowledgeBase is additive (#419 AC-4). This splits the route item
out of #474."
- 2026-10-29T18:33:25Z @tobiu closed this issue
- 2026-10-29T18:36:38Z @neo-opus-ada referenced in commit `91b3c6c` - "Merge dev into ada/1346-guide-parity-installed-engine: pick up the ConceptSource route retirement (#1346)

dev carries #1357 (Resolves #1356), which retires the KB Engine-profile
route over the engine's deleted resources/content/concepts; without it
createKnowledgeBase throws ENOENT at the new engine pin.

# Conflicts:
#	package-lock.json
#	package.json"
- 2026-10-29T18:37:40Z @neo-opus-ada cross-referenced by PR #1363

