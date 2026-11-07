---
id: 1515
title: Hydrate multi-chunk and Engine references in SearchService#ask
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-07T03:12:10Z'
updatedAt: '2026-11-07T04:04:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1515'
author: neo-gemini-maryam
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
# Hydrate multi-chunk and Engine references in SearchService#ask

## Context

Promoted from `@neo-fable-clio`'s defect note (`MESSAGE:78a07f7b-de12-4d90-b647-91e5e05f1059`, flagged by `@tobiu` during the `v13.4.0` front-door verification pass):

- Calling `ask_knowledge_base({ query: "Which component families and classes does Neo.mjs ship — grid, form fields, calendar, dashboard, layouts, main-thread addons? List them with counts.", type: "guide" })` returns `"I don't have enough information"` for `Grid`, `Calendar`, `Dashboard`, and component-family counts, even though its own `#2` ranked reference (`learn/guides/fundamentals/CodebaseOverview.md`, score `18390`) contains `### Component Library (The "Batteries")` at lines `255–290` (`offset 12692..14048`) and `#1` (`learn/Glossary.md`, score `20452`) contains `## Component Library`.
- Worse, `ask_knowledge_base` emits **no** truncation notice (`assembled.truncated === false`) on that response, making the partial 1-chunk context look like the complete 5-document corpus was consulted.

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1297`–`#1513`) at `2026-11-07T03:11:08Z`; no equivalent found.
- A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-11-07T03:11:06Z`; no competing claim on `QueryService` / `SearchService` hydration.
- MC sweep: `query_raw_memories({ query: "ask_knowledge_base hydrateReferenceContent sourceMetadata multi-chunk CodebaseOverview I don't have enough information", nResults: 5 })`, 5 results, no prior decision found.
- Own-assignment sweep: 0 open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-brain`, none overlapping.
- Structure map gate: `npm run ai:structure-map -- --files --loc` verified owning files `ai/services/knowledge-base/QueryService.mjs` (`570` codeLoc), `ai/services/knowledge-base/SearchService.mjs` (`326` codeLoc), and `ai/services/knowledge-base/helpers/askContextBudget.mjs` (`60` codeLoc).

## The Problem

Empirical measurement of `QueryService.queryDocuments` + `SearchService.hydrateReferenceContent` on the reproduction query exposes a three-layer failure chain:

1. **Single-chunk `sourceMetadata` overwrite in `QueryService#queryDocuments` (`ai/services/knowledge-base/QueryService.mjs:180-182`):**
   `DocumentationParser` splits `learn/guides/fundamentals/CodebaseOverview.md` (`42,307` chars) into `55` per-heading chunks in ChromaDB, and `SourceParser` splits `.mjs` class files into per-method/config chunks. When ChromaDB returns `nResults` (`100`) chunks, `15` chunks of `CodebaseOverview.md` match and `QueryService` sums their scores into `sourceScores[sourcePath]` (`18390`). However, `if (!sourceMetadata[sourcePath]) { sourceMetadata[sourcePath] = metadata; }` keeps **only the single first chunk's `metadata.content`** (`Codebase Overview - Why This Matters for You`, `986` chars) and discards the other `14` matched chunks (`6,265` chars). Every reference hydrated via `metadata.content` (including all revision-bound Engine profile rows with `rootKind === 'neo-workspace' && metadata.extractionIdentity` and all non-local tenant rows) therefore feeds synthesis only a single chunk (`196` chars for `Glossary.md`, `986` chars for `CodebaseOverview.md`, `728` chars for `Forms.md`, `466` chars for `ComponentsAndContainers.md`).
2. **Missing `node_modules/neo.mjs/` fallback for unprefixed Engine paths in `SearchService#hydrateReferenceContent` (`ai/services/knowledge-base/SearchService.mjs:393-405`):**
   In `neo-agent-brain`, `aiConfig.neoRootDir` is the Brain checkout root while Engine guides (`learn/Glossary.md`, `learn/guides/...`, `learn/tutorials/...`), classes (`src/...`), and examples (`examples/...`) live under `<neoRootDir>/node_modules/neo.mjs/`. Legacy/synced ChromaDB rows store `ref.source` without the `node_modules/neo.mjs/` prefix (`rootKind: null`), so `path.resolve(localRoot, ref.source)` checks `<neoRootDir>/learn/guides/fundamentals/CodebaseOverview.md` (`existsAtRoot: false`, `existsInNodeModules: true`) and falls back to the single-chunk `metadata.content`.
3. **Blind head-slice (`raw.slice(0, 12000)`) on oversized multi-section documents (`ai/services/knowledge-base/SearchService.mjs:634-661` / `ai/services/knowledge-base/helpers/askContextBudget.mjs:96`):**
   Even when `node_modules/neo.mjs/learn/guides/fundamentals/CodebaseOverview.md` (`42,307` chars) is read from disk, `assembleAskContext` slices characters `0..12000` (`contextMaxCharsPerDocument = 12000`) from the top of the file. In `CodebaseOverview.md`, `### Component Library (The "Batteries")` sits at characters `12,692..14,048` (`lines 255–290`), so a blind head slice cuts off the exact section answering the query right before line `255`.

## The Architectural Reality

- `Design authority:` `ai/services/knowledge-base/SearchService.mjs:150-152` (*"1. Retrieves: Uses `QueryService` to find the most relevant files based on semantic similarity and intelligent scoring... 2. Reads: Fetches the full content of these files from the local filesystem to ensure the LLM has complete context (avoiding truncated metadata)."*) and `ai/services/knowledge-base/QueryService.mjs:381-386` (*"Keep distinct chunks from the same source, but avoid double-counting the same metadata row."*).
- `QueryService.mjs:164-183` builds `sourceMetadata[sourcePath]` while iterating `metadatas` in relevance order, and `SearchService.mjs:367-417` (`hydrateReferenceContent`) resolves either embedded metadata (`isNonLocalTenantReference` or `rootKind === 'neo-workspace' && metadata.extractionIdentity`) or local filesystem bytes before `assembleAskContext` (`askContextBudget.mjs:75-141`) enforces `contextMaxCharsPerDocument` (`12,000`) and `contextBudgetChars` (`48,000`).

## The Fix

1. **Multi-chunk `sourceMetadata` aggregation (`ai/services/knowledge-base/QueryService.mjs`):**
   Track distinct non-empty `metadata.content` chunks matching the selected document's provenance (`tenantId`, `repoSlug`, `rootKind`, `extractionIdentity`) per `sourcePath` in relevance order during the `metadatas` scoring pass (`sourceChunks[sourcePath]`). Preserve the highest-ranked chunk's metadata object on `sourceMetadata[sourcePath]`, and when multiple distinct chunks match for that document's provenance, join their `content` blocks in relevance order (`\n\n`) onto `sourceMetadata[sourcePath].content` (while keeping single-chunk `sourceMetadata[sourcePath]` byte-identical and never mixing chunks across distinct repository, revision, or tenant identities).
2. **Installed Engine package fallback in `SearchService#hydrateReferenceContent` (`ai/services/knowledge-base/SearchService.mjs`):**
   When hydrating a local default-tenant reference (`!isNonLocalTenantReference(metadata)` and `!(metadata.rootKind === 'neo-workspace' && metadata.extractionIdentity)`) whose `ref.source` is relative and does not exist at `path.resolve(localRoot, ref.source)`, check `path.resolve(aiConfig.neoRootDir, 'node_modules/neo.mjs', ref.source)` (bounded within `path.resolve(aiConfig.neoRootDir, 'node_modules/neo.mjs')`) before falling back to `embeddedContent`.
3. **Fence-aware, query-prioritized section ordering for oversized multi-section documents (`ai/services/knowledge-base/SearchService.mjs`):**
   When a hydrated document's content exceeds `maxCharsPerDocument` (`> 0`) and contains multiple Markdown heading sections outside fenced code blocks (`splitMarkdownSections`), reorder its sections in descending order of relevance to `(query, embeddedContent)` (with original section index as tie-breaker) while preserving fenced code blocks intact with their enclosing heading/warning prose and preserving total character length so `assembleAskContext` still truncates at `maxCharsPerDocument` and still emits `assembled.truncated = true` + `assembled.notice`. Documents within `maxCharsPerDocument` (or single-section documents) remain untouched and byte-identical.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `QueryService#queryDocuments` (`includeMetadata: true`) | `ai/services/knowledge-base/QueryService.mjs:164-183` | Aggregates distinct non-empty `metadata.content` chunks matching the selected document's provenance (`tenantId`, `repoSlug`, `rootKind`, `extractionIdentity`) per `sourcePath` in relevance order into `result.metadata.content` while preserving top-chunk metadata fields. | Single-chunk and lexical-rescue sources retain existing `metadata` shape unchanged; chunks with differing provenance are not mixed. | Yes (JSDoc) | `test/playwright/unit/ai/services/knowledge-base/QueryService.queryDocuments.spec.mjs` |
| `SearchService#hydrateReferenceContent` | `ai/services/knowledge-base/SearchService.mjs:367-417` | Resolves unprefixed relative default-tenant Engine paths under `<neoRootDir>/node_modules/neo.mjs/<ref.source>` when absent at `<localRoot>/<ref.source>`; prioritizes query-relevant Markdown sections using fence-aware section boundaries when hydrated content exceeds `maxCharsPerDocument`. | Non-local tenant rows and revision-bound Engine profile rows (`rootKind: 'neo-workspace'` + `extractionIdentity`) continue to use `metadata.content` without touching the filesystem; missing files fall back to `embeddedContent` then `'No Content (File missing or empty)'`. | Yes (JSDoc) | `test/playwright/unit/ai/services/knowledge-base/SearchService.spec.mjs` |

## Decision Record Impact

`none` (restores the documented `QueryService` + `SearchService` RAG hydration contract while preserving `ADR-0019` `AiConfig` read-at-use-site rules and `#16999` context-budget invariants).

## Acceptance Criteria

- [ ] `QueryService.queryDocuments({ ..., includeMetadata: true })` joins all distinct matched chunk `content` strings matching the selected document's provenance (`tenantId`, `repoSlug`, `rootKind`, `extractionIdentity`) per `sourcePath` in relevance order onto `result.metadata.content` instead of discarding every matched chunk after the first or mixing bytes across distinct document identities.
- [ ] `SearchService.hydrateReferenceContent(ref)` falls back to `<neoRootDir>/node_modules/neo.mjs/<ref.source>` for local default-tenant relative paths not found directly under `<localRoot>/<ref.source>`, without permitting path traversal or filesystem fallback for non-local tenant or revision-bound (`rootKind: 'neo-workspace'` + `extractionIdentity`) rows.
- [ ] When a hydrated multi-section Markdown document exceeds `contextMaxCharsPerDocument` (`12,000` chars), `SearchService` prioritizes sections matching the query and matched chunks ahead of generic preamble sections using fence-aware section boundaries while preserving fenced code blocks with their enclosing heading/warning prose and preserving total document length so `assembleAskContext` still enforces `contextMaxCharsPerDocument` and appends the truncation notice.
- [ ] Unit tests in `QueryService.queryDocuments.spec.mjs` and `SearchService.spec.mjs` cover multi-chunk `metadata.content` aggregation (including repository, revision, and tenant collision controls), `node_modules/neo.mjs/` fallback hydration, and fence-aware oversized multi-section guide prioritization with `assembled.notice` preservation.

## Out of Scope

- Changing `contextBudgetChars` (`48,000`) or `contextMaxCharsPerDocument` (`12,000`) defaults in `ai/mcp/server/knowledge-base/configBase.mjs`.
- Re-ingesting or rebuilding the distributed ChromaDB release archive.

## Avoided Traps

- **Raising `contextMaxCharsPerDocument` to `50,000+`:** Would blow past the per-document context budget that `#16999` introduced to prevent a single 42KB–50KB guide (`CodebaseOverview.md`, `Earthquakes.md`) from starving documents 2–5.
- **Dropping non-matching sections before `assembleAskContext`:** If `hydrateReferenceContent` pre-shrunk a `42,307`-char guide to `< 12,000` chars before calling `assembleAskContext`, `assembleAskContext` would see `raw.length <= perDocCap` and omit the mandatory truncation notice (`Context note: Document(s) ... were shortened`). Reordering sections while keeping all sections in the string preserves the exact byte length (`42,307 > 12,000`) so `assembleAskContext` still performs the `12,000`-char cut and emits `assembled.notice`.

## Related

- Promoted from `MESSAGE:78a07f7b-de12-4d90-b647-91e5e05f1059` (`@neo-fable-clio` defect note)
- Context budget baseline: `neomjs/neo#16999`

Origin Session ID: 444579f0-7e38-4519-831e-f7e5aabe8f44
Retrieval Hint: "ask_knowledge_base CodebaseOverview hydrateReferenceContent sourceMetadata multi-chunk Component Library The Batteries"

## Timeline

- 2026-11-07T03:12:11Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-07T03:12:12Z @neo-gemini-maryam added the `bug` label
- 2026-11-07T03:12:12Z @neo-gemini-maryam added the `ai` label
- 2026-11-07T03:12:12Z @neo-gemini-maryam added the `architecture` label
- 2026-11-07T03:12:12Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-07T03:23:26Z @neo-gemini-maryam cross-referenced by PR #1516
- 2026-11-07T04:03:54Z @neo-gemini-maryam referenced in commit `571d3d4` - "fix(knowledge-base): preserve document provenance and fenced sections (#1515)"
- 2026-11-07T04:14:52Z @neo-gemini-maryam referenced in commit `189e5e2` - "fix(knowledge-base): escape backtick in fence matcher for retry-bounds scanner (#1515)"

