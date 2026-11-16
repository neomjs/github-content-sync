---
id: 1806
title: Stop unconditional guide bonus and Base.mjs boost from burying source
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-16T14:31:24Z'
updatedAt: '2026-11-16T19:46:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1806'
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
# Stop unconditional guide bonus and Base.mjs boost from burying source

## Context

Measured in `#1800` across the live `92,257`-chunk `neo-knowledge-base` Chroma collection (`openAiCompatible` / `text-embedding-qwen3-embedding-8b`, `neomjs/neo` `459371e353`, `neomjs/neo-agent-brain` `a2a621cb78`) using 49 real agent queries from `kb_query_log` (`ai/scripts/diagnostics/fixtures/kbRankingQuestions.json`):

- On the pre-registered balanced 40-question set (`20` Engine + `20` Brain), shipped `QueryService#queryDocuments` leads overall (`26/40 = 65%` Hit@5, median found rank `2`, MRR `0.481`) over unweighted nearest-chunk Chroma retrieval (`15/40 = 38%`, MRR `0.259`) and checkout `ripgrep` (`10/40 = 25%`, MRR `0.132`).
- Across all 49 verified questions, however, `QueryService` **loses** to plain unweighted nearest-chunk Chroma retrieval on `source-with-guide-mention` (`10/18 = 56%` Hit@5 vs `11/18 = 61%` for nearest chunks) and reaches only `4/11 (36%)` Hit@5 on `source-only` questions (with `0/3` answered by `ask_knowledge_base`).

## The Problem

Three compounding defects in `ai/services/knowledge-base/QueryService.mjs` let unrelated guides and `src/core/Base.mjs` outrank the source files that own an answer, and prevent lexical rescue from reaching installed Engine files:

1. **Unconditional per-word `guideMatch` bonus (`E107`, `E126`, `E183`, `E022`, `B056`):**
   Every word longer than two characters in `queryWords` adds `queryScoreWeights.guideMatch` (`+50`) to *every* candidate chunk whose `metadata.type === 'guide'`, regardless of whether that word appears in the guide's path, title, or content. A 9–14 word query hands every indexed guide a flat `+450` to `+700` bonus.
   - Observed demotions against unweighted nearest chunks:
     - `E107` (`src/core/Config.mjs`): nearest chunks `#4` → shipped `#9`
     - `E126` (`src/data/Model.mjs` / `src/collection/Filter.mjs`): nearest chunks `#3` → shipped `#9`
     - `E183` (`src/data/TreeStore.mjs`): nearest chunks `#3` → shipped `#6` (with `KnowledgeBaseMcpApi.md` and `MemoryCoreMcpApi.md` lifted into top 4)
     - `B056` (`ai/mcp/server/shared/services/RuntimeFreshnessService.mjs`): nearest chunks `#2` → shipped `#11`
     - `E022` (`src/grid/Row.mjs`): nearest chunks `#6` → shipped `#14`
2. **Unbounded `src/core/Base.mjs` `baseFileBonus` + `inheritanceBoost` accumulation:**
   Because Chroma returns multiple chunks per subclass in candidate pools, `src/core/Base.mjs` accumulates `baseFileBonus` (`+20`) on its own chunks plus `inheritanceBoost` (`+80`) from every class candidate whose `inheritanceChain` terminates at `src/core/Base.mjs`. Across the 49 queries, `src/core/Base.mjs` ranked `#1` on `12/49` questions — including pure Brain questions (`B004`, `B015`, `B056`, `B112`, `B116`, `B117`).
3. **Brain-only lexical rescue root (`getLexicalRescueCandidates`):**
   Lexical rescue only resolves paths against `aiConfig.neoRootDir`, whereas `SearchService#hydrateReferenceContent` (`ai/services/knowledge-base/SearchService.mjs:413`) falls back to `node_modules/neo.mjs/` for Engine paths. Post-sync Engine files (`E132` `src/main/addon/Clipboard.mjs`, `E066` `src/gantt/DependencyStore.mjs`) never benefit from path/filename/guide-title rescue even though `ripgrep` finds them.

## The Architectural Reality

Design authority: "QueryService ranks semantic matches, boosts source authority, applies inheritance context, and keeps historical artifacts available without letting them dominate current-source questions." (`learn/agentos/KnowledgeBase.md:156`)

- `ai/services/knowledge-base/QueryService.mjs`:
  - Lines `198-222`, specifically line `212` (`if (metadata.type === 'guide') score += queryScoreWeights.guideMatch;` inside `queryWords.forEach`), unconditionally adds `guideMatch` per query word without checking match against guide path/title/name/chunk content.
  - Lines `227` (`baseFileBonus`) and `233-240` (`inheritanceChain.forEach`) accumulate unweighted base/inheritance boosts linearly with candidate chunk count rather than bounding per-source contribution or discounting root `src/core/Base.mjs` accumulation.
  - Lines `641-644`, `676`, `713`, `744`, `807` in `getLexicalRescueCandidates` and its helpers check only `aiConfig.neoRootDir`.
- `ai/mcp/server/knowledge-base/configBase.mjs:834-851` defines `queryScoreWeights`.
- `ai/scripts/diagnostics/measureKbRanking.mjs` and `ai/scripts/diagnostics/fixtures/kbRankingQuestions.json` establish the reproducible 40-question and 49-question baseline.

## The Fix

1. **Gate `guideMatch` on actual relevance (not flat query word count):** Apply `guideMatch` at most once per chunk (or scaled by actual matches against `sourcePathLower`, `fileName`, `nameLower`, or chunk content) so unrelated guides no longer outrank exact source files on multi-word technical queries.
2. **Bound `inheritanceBoost` / `src/core/Base.mjs` accumulation:** Cap repeated per-chunk accumulation onto the same ancestor source (or attenuate root `src/core/Base.mjs` / decay per distinct class rather than summing across every chunk of every subclass).
3. **Extend lexical rescue resolution to `node_modules/neo.mjs/`:** Mirror `SearchService#hydrateReferenceContent` (`SearchService.mjs:413-422`) when checking local file existence, `learn/tree.json`, path/filename hints, and code-term rescue so Engine files are rescue-eligible alongside Brain files.
4. **Re-run the `#1800` benchmark before merge:** Run `node ai/scripts/diagnostics/measureKbRanking.mjs --all-49` and record the before/after delta on `source-with-guide-mention` and `source-only` without regressing `parent-class` or `guide-covered`.

Decision Record impact: none.

## Acceptance Criteria

- [ ] `QueryService#queryDocuments` no longer awards `queryScoreWeights.guideMatch` per unmatched query word; guides matched only by candidate pool membership do not receive an unconditional `queryWords.length * 50` boost.
- [ ] `src/core/Base.mjs` boost accumulation across chunks/subclasses is bounded so unrelated queries (including Brain `B*` queries) do not rank `src/core/Base.mjs` `#1`.
- [ ] `QueryService#getLexicalRescueCandidates` (path hints, filename hints, and Engine guide titles) resolves `node_modules/neo.mjs/` alongside `aiConfig.neoRootDir`.
- [ ] Unit tests cover each regression (`source-with-guide-mention` source beating unmatching guide, bounded `Base.mjs` accumulation, `node_modules/neo.mjs/` lexical rescue).
- [ ] Re-running `node ai/scripts/diagnostics/measureKbRanking.mjs --all-49` improves `source-with-guide-mention` and `source-only` Hit@5 over the `#1800` baseline (`10/18` and `4/11`) without regressing `parent-class` (`4/5`) or `guide-covered` (`10/15`).

## Out of Scope

- Changing `SearchService#ask` oversized file windowing (tracked in sibling ticket filed from `#1800`).
- Re-syncing the local primary-repo Chroma corpus or adding neural rerankers.

## Avoided Traps

- **Dropping `inheritanceBoost` completely:** `#1800` proved inheritance boost drives `parent-class` from `1/5` (`20%`, nearest chunks) to `4/5` (`80%`, `3/3` strict); preserve ancestor boost while capping `src/core/Base.mjs` runaway accumulation.
- **Tuning weights in `configBase.mjs` without fixing the loop bug:** Lowering `guideMatch` still scales linearly with `queryWords.length` as long as `if (metadata.type === 'guide')` ignores whether the word matched the guide.

## Related

- Related: #1800, neomjs/neo#21089, neomjs/neo#21093

unowned-rationale: Filed as AC-4 follow-up from `#1800` measurement; actionable unit-tested fix inside `QueryService.mjs` with reproducible before/after harness already committed.

Live latest-open + closed sweep: checked latest 25 open of 156 (red) + open/closed search (`QueryService OR guideMatch OR inheritanceBoost OR assembleAskContext OR lexicalRescue`) + A2A last 30 + MC sweep + own-assignment sweep (`2` open: `#1800`, `#1764`) on 2026-11-16T14:22Z–14:29Z; no equivalent open or closed ticket found.

Origin Session ID: 76097b8c-6818-4c1f-a0c1-abf162c6100c

Retrieval Hint: "QueryService guideMatch per query word Base.mjs inheritanceBoost source-only source-with-guide-mention ranking regression"

🧭 Maryam · @neo-gemini-maryam

## Timeline

- 2026-11-16T14:31:25Z @neo-gemini-maryam added the `bug` label
- 2026-11-16T14:31:25Z @neo-gemini-maryam added the `good first issue` label
- 2026-11-16T14:31:26Z @neo-gemini-maryam added the `help wanted` label
- 2026-11-16T14:31:26Z @neo-gemini-maryam added the `ai` label
- 2026-11-16T14:31:26Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-16T14:31:39Z @neo-gemini-maryam cross-referenced by #1807
- 2026-11-16T16:28:57Z @neo-fable-clio cross-referenced by PR #1808
- 2026-11-16T20:01:41Z @neo-gemini-maryam cross-referenced by PR #1820

