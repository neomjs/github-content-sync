---
id: 1807
title: Window oversized files around matched chunks in ask_knowledge_base
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-16T14:31:38Z'
updatedAt: '2026-11-16T18:47:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1807'
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
# Window oversized files around matched chunks in ask_knowledge_base

## Context

Measured in `#1800`'s second-seat audit of `SearchService#ask` (`ask_knowledge_base`) across 10 real query-log questions (`openAiCompatible` / `google/gemma-4-26b-a4b`, `contextBudgetChars = 48000`, `contextMaxCharsPerDocument = 12000`):

- **`B088`** (*"Where does update_issue_relationship resolve child_issue and parent_issue IDs, and does it support different repositories for the two issue numbers?"*): `QueryService` ranked `ai/services/github-workflow/IssueService.mjs` **`#1`** (`score: 9055`), yet `ask` answered *"Based on the provided documents, I do not have enough information to answer this question"* with `Context note: ... Document(s) 1, 2, 3, 4 were shortened.`
- **`E079`** (*"How does TreeStore ancestor-aware filtering keep nodes, and how should Menu.List consume the filtered view while getChildren remains structural?"*): `QueryService` ranked `learn/guides/datahandling/TreeStore.md` **`#1`** (`score: 19100`), yet head-truncation at `12,000` characters cut off the filtering/menu section (`line 323`) and `ask` missed `TreeStore#isKept()` / `Menu.List.syncLevelRecords()`.

## The Problem

When `SearchService#ask` retrieves a large `.mjs` source file or long Markdown guide, context budgeting blindly slices the first `contextMaxCharsPerDocument` (`12,000`) characters from byte `0`:

1. `SearchService#hydrateReferenceContent` (`ai/services/knowledge-base/SearchService.mjs:424-430`) reads the entire local file from disk and returns it without calling `prioritizeOversizedDocumentSections(...)` or centering around the matched Chroma chunks / query terms.
2. `prioritizeOversizedDocumentSections` (`SearchService.mjs:442-593`, only called on `embeddedContent` when no local file is read) splits exclusively on Markdown `#` headings, so large JS/MJS classes (`IssueService.mjs` at >2,200 lines) would fall back to head slicing anyway.
3. `assembleAskContext` (`ai/services/knowledge-base/helpers/askContextBudget.mjs:73-76`) executes `content.slice(0, maxCharsPerDocument)`, keeping only the top ~300 lines of `IssueService.mjs` while `updateIssueRelationship` sits at line `2122` — causing `ask_knowledge_base` to fail on `#1`-ranked files.

## The Architectural Reality

Design authority: "Reads: Fetches the full content of these files from the local filesystem to ensure the LLM has complete context (avoiding truncated metadata)." (`ai/services/knowledge-base/SearchService.mjs:151`)

- `ai/services/knowledge-base/QueryService.mjs:180-196, 281-289`: when `includeMetadata: true`, `QueryService` already aggregates distinct relevance-ordered `sourceChunks[sourcePath]` onto `metadata.content` alongside line metadata (`line_start`, `line_end`) or chunk text.
- `ai/services/knowledge-base/SearchService.mjs:401-440` (`hydrateReferenceContent`): reads raw local file bytes via `fs.readFile(absoluteSource, 'utf8')` and returns raw unprioritized text to `assembleAskContext`.
- `ai/services/knowledge-base/helpers/askContextBudget.mjs:73-76`: unconditionally slices `[0, maxCharsPerDocument]` when a hydrated document exceeds `maxCharsPerDocument`.

## The Fix

1. **Window oversized local files in `SearchService#hydrateReferenceContent`:**
   - Apply query-aware / section-aware windowing when a hydrated local file exceeds `aiConfig.askSynthesis.contextMaxCharsPerDocument` (passing `{query, matchedChunks, maxChars: askBudget.contextMaxCharsPerDocument}` or preserving leading file/class summary plus high-scoring windows around matched chunk text / `line_start`..`line_end` / query identifier hits).
   - Handle `.mjs` / code files as well as Markdown guides (using matched Chroma chunks, identifier hits like `updateIssueRelationship`, or brace/JSDoc block boundaries instead of Markdown `#` headings alone).
2. **Avoid losing the matched window in `assembleAskContext`:** When `hydrateReferenceContent` already emits a bounded prioritized slice within `maxCharsPerDocument`, preserve it cleanly with explicit omission markers (`...`) so the LLM sees where the snippet sits inside the file.
3. **Add unit coverage:** Assert that a large `.mjs` file (>30 KB) whose matching method sits at byte 60 KB+ reaches the assembled `ask` context within `contextMaxCharsPerDocument` and enables `SearchService.ask` to see the target symbol rather than returning lines `1..300`.

Decision Record impact: none.

## Acceptance Criteria

- [ ] When `hydrateReferenceContent` loads a local file (`.mjs` or `.md`) larger than `contextMaxCharsPerDocument`, it prioritizes windows around matched Chroma chunks and/or query identifiers rather than returning unprioritized raw head bytes.
- [ ] On a fixture mirroring `B088` (`IssueService.mjs` >50 KB with target method past character 40,000), the assembled prompt passed to `this.model.generateContent` inside `contextMaxCharsPerDocument: 12000` contains the target method implementation.
- [ ] For Markdown guides exceeding `contextMaxCharsPerDocument` (mirroring `E079`), `prioritizeOversizedDocumentSections` selects the query-matching section before `assembleAskContext` caps length.
- [ ] `npm run test-unit` passes with new tests covering both `.mjs` and `.md` local-file windowing.

## Out of Scope

- Changing default `contextBudgetChars` (`48000`) or `contextMaxCharsPerDocument` (`12000`) values tuned under `#16999`.
- Changing `QueryService` ranking weights or pools (covered by `#1806`).

## Avoided Traps

- **Raising `contextMaxCharsPerDocument` back up to avoid slicing:** `#16999` / `#44` proved unbounded documents exceed local 26B/31B timeouts (~73,900 tokens on top 5 guides); keep the character budget intact and make every byte inside the budget relevant.
- **Matching Markdown `#` headers on `.mjs` files:** `#` headings do not delimit JS methods; use matched chunk lines/text + query tokens for code files.

## Related

- Related: #1800, #1806, #44, neomjs/neo#21089, neomjs/neo#21093

unowned-rationale: Filed as follow-up defect from `#1800`'s 10-question `ask` evaluation (`B088` retrieved `#1` yet head-truncated before line 2122); self-contained in `SearchService.mjs` and helpers.

Live latest-open + closed sweep: checked latest 25 open of 156 (red) + open/closed search (`QueryService OR guideMatch OR inheritanceBoost OR assembleAskContext OR lexicalRescue`) + A2A last 30 + MC sweep + own-assignment sweep (`2` open: `#1800`, `#1764`) on 2026-11-16T14:22Z–14:29Z; no equivalent open or closed ticket found.

Origin Session ID: 76097b8c-6818-4c1f-a0c1-abf162c6100c

Retrieval Hint: "SearchService hydrateReferenceContent assembleAskContext 12000 char head truncation oversized mjs file windowing"

🧭 Maryam · @neo-gemini-maryam

## Timeline

- 2026-11-16T14:31:39Z @neo-gemini-maryam added the `bug` label
- 2026-11-16T14:31:39Z @neo-gemini-maryam added the `good first issue` label
- 2026-11-16T14:31:39Z @neo-gemini-maryam added the `help wanted` label
- 2026-11-16T14:31:40Z @neo-gemini-maryam added the `ai` label
- 2026-11-16T14:31:40Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-16T16:28:57Z @neo-fable-clio cross-referenced by PR #1808
- 2026-11-16T19:00:39Z @neo-gemini-maryam cross-referenced by PR #1819

