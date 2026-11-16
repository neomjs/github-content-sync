---
id: 1800
title: Measure the Knowledge Base ranking against plain retrieval and grep
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-16T08:31:34Z'
updatedAt: '2026-11-16T18:01:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1800'
author: neo-fable-clio
commentsCount: 2
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
closedAt: '2026-11-16T18:01:28Z'
---
# Measure the Knowledge Base ranking against plain retrieval and grep

## Context

Part 3 of the memory series (neomjs/neo#21093) describes the Knowledge Base's ranking with its weights. On 16 Nov 2026 the operator asked whether a stranger would be excited and try it. Two cold reads of the post by a reader with no session context answered no, and both named the same missing thing unasked: "the weights look hand-tuned; show they matter", and "the same 30 questions answered by grep, plain nearest-chunk retrieval, and your ranking, scored by wrong-answer rate. Right now the weights rest on one anecdote."

Observed: no measurement of the ranking exists in this repository's issues (searches below). Recalled, not re-verified: on 11 Oct 2026 a turn of mine planned to move from qualitative feedback to recall@5 before any reranking work; no ticket or instrument came of it.

## The Problem

`QueryService.queryDocuments` draws 100 candidates in four pools (65 primary, 20 secondary, 10 historical, 5 unfiltered) and scores them with weights set by hand: a ticket loses 70, a pull request 250, a guide gains 50 for each word of the question, a parent class file gains 80. Ten maintainers use the result all day and it serves them. Nobody can say by how much it beats the two things a reader would build first, nearest chunks and grep, or on which kinds of question it loses.

One suspect is on record: the guide bonus counts every word of the question longer than two characters, matched or not, so a long question lifts every guide.

## The Architectural Reality

- `ai/services/knowledge-base/QueryService.mjs`, `queryDocuments` (the pools and the scoring), with its weights in `ai/mcp/server/knowledge-base/configBase.mjs` (`queryScoreWeights` and the pool shares).
- `ai/services/knowledge-base/SearchService.mjs`, `ask`: the top five files read whole, one grounded answer.
- `list_agent_faqs` returns questions agents actually asked; Memory Core holds more. They are the honest source of test questions, since questions written by the ranking's authors fit the ranking.

## The Fix

One diagnostic, three arms, one committed table:

1. **A fixture of at least 30 real questions**, each with the file or files that hold its answer, verified by a seat that did not pick the question. Half about the engine, half about the Brain; at least five whose answer lives in a parent class and five whose answer is in source with no guide.
2. **Three arms on the same index and checkout:** (a) the shipped ranking through `queryDocuments`; (b) plain nearest chunks from the same collection with no pools and no scoring, folded to files; (c) `ripgrep` over the checkout with the question's content words, in two orderings: files ranked by distinct words matched, then by match count; and by summed match count alone, the first prescription, kept as the pre-registered variant. *(Amended 16 Nov 2026 after the review of #1808: summed count alone ranks a long file that repeats one common word above the file that defines the symbol, and scores 10/40 where the first ordering scores 30/40.)*
3. **Two numbers per arm:** how often an answer file is in the top five, and the rank of the first answer file. For ten of the questions, the answer of `ask` is judged right or wrong against the fixture by a second seat.
4. **The table lands in `learn/agentos/KnowledgeBase.md`** (or its measurements page) with the fixture's path, the index size and the commit. A class of question the ranking loses becomes its own ticket.

## Acceptance Criteria

- [ ] AC-1 The fixture is committed with its provenance (where each question came from, who verified the answer files).
- [ ] AC-2 The diagnostic runs the three arms and prints the table; its command is in the guide.
- [ ] AC-3 The guide carries the table with index size, date and commit.
- [ ] AC-4 Every class of question where the shipped ranking is not the best arm has a ticket, or the table says there is none.
- [ ] AC-5 (post-merge) The memory series' part 3 cites the number in place of the single June case.

## Out of Scope

- Changing any weight. This ticket measures.
- A repository other than ours. A tenant arm is a later leaf once this instrument exists.
- Rerankers and hybrid retrieval.

## Avoided Traps

- **Questions written for the test.** They fit the ranking. Use the questions agents asked.
- **Only a judged answer rate.** A model judging a model hides which stage failed; the file ranks come first.
- **One mixed number.** The classes (parent class, source only, history) are reported apart, because the ranking's rules target them apart.

## Related

neomjs/neo#21093, neomjs/neo#21089 (the post) · neomjs/neo-agent-skills#409 (the review verdict that asked the question)

Decision Record impact: none.

unowned-rationale: needs a seat with access to the plane's index and budget this week; mine is spent on the launch window and the film. The fixture's design read is mine on request. Not `good first issue`: an outside contributor has no index of this size. Part 3 of the memory series is held until this table exists: the operator's bar of 16 Nov is a cold read of 9 of 10 or better, the post stands at 6, and all three cold reads asked for this measurement.

Live latest-open + closed sweep: latest 20 open of 158 (red) and three searches across the organization ("knowledge base ranking benchmark OR eval OR recall OR precision", "QueryService weights measured comparison retrieval baseline", "recall@5") at 2026-11-16T08:40Z: no equivalent and no owning ticket or epic. A2A claim sweep: this session's mailbox pages through 08:11Z, no claim on the ranking. MC sweep: "knowledge base retrieval quality measured: ranking weights versus plain vector similarity or grep", 5 results: my turn of 11 Oct (recall@5 planned) and Julia's audit of 16 Oct (hybrid retrieval proposed); no measurement. Own-assignment sweep: none on this surface. Structure map: not run; the diagnostic's home is the implementer's placement call under `ai/scripts/diagnostics/`.

Origin Session ID: 8242bf57-a3e6-4237-947a-661ed4ea3590

Retrieval Hint: "knowledge base ranking measured against nearest chunks and grep, recall at five, real agent questions fixture"

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code · session 8242bf57-a3e6-4237-947a-661ed4ea3590

## Timeline

- 2026-11-16T08:31:36Z @neo-fable-clio added the `enhancement` label
- 2026-11-16T08:31:36Z @neo-fable-clio added the `ai` label
- 2026-11-16T08:31:36Z @neo-fable-clio added the `testing` label
- 2026-11-16T08:31:37Z @neo-fable-clio added the `agent-os` label
- 2026-11-16T08:33:35Z @neo-fable-clio cross-referenced by PR #21093
- 2026-11-16T08:53:22Z @neo-fable-clio cross-referenced by #21089
### @neo-fable-clio - 2026-11-16T10:15:05Z

**Routing state, Mon 16 Nov, 10:45Z.** Julia declined for the Search Console priority (her message of 09:29Z). Maryam took the verifier and three-arm half at 09:52Z (her message; the first version of this comment, written at 10:15Z, had not read it and named a GPT seat as fallback, which is void). The split: I pick the fixture's questions with candidate answer files, from the Knowledge Base's own query log; Maryam verifies the answer files and builds the three arms. That keeps the rule that the seat that picks a question does not verify it.

The fixture draft follows as its own comment. The dependent pull request neomjs/neo#21093 stays a draft until the table exists.

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code

### @neo-fable-clio - 2026-11-16T11:10:10Z

**Fixture draft: 49 questions with candidate answer files.** Picker: Clio. Verifier: the seat that builds the arms. Nothing below is verified yet; that is the next step by rule.

| Repository | Questions | Asked as a sentence | Asked as keywords | parent-class | source-only | source-with-guide-mention | guide-covered |
|---|---:|---:|---:|---:|---:|---:|---:|
| `neomjs/neo` | 29 | 14 | 15 | 5 | 4 | 10 | 10 |
| `neomjs/neo-agent-brain` | 20 | 12 | 8 | 0 | 7 | 8 | 5 |
| Sum | 49 | 26 | 23 | 5 | 11 | 18 | 15 |

Judged subset for `ask`: 10 questions carry a `referenceAnswer`.

**Source.** The Knowledge Base's own query log (`kb_query_log`, read through `list_agent_faqs` with `refresh`, `minCount: 1`, `limit: 300` at 2026-11-16T10:20Z): 300 distinct queries, 58 of them asked two to ten times between 1 Aug and 11 Nov, 242 asked once between 11 and 16 Nov. The wider rebuild of the FAQ table heals itself: the REM pass rebuilds it with its defaults (`GapInferenceEngine#getKbDemandGapsByConcept`).

**Pick rule, fixed before any answer file was looked up.** In list order (count descending, then newest first), the first 24 eligible queries per repository. Eligible: the answer can live in a file of `neomjs/neo` (`src/`, `learn/`, `apps/`, `buildScripts/`) or of this repository (`ai/`, `learn/`); a keyword query names at least one symbol or file; of two rephrasings from one turn, the first. Not eligible: history questions ("which ticket …"), because the grep arm has no ticket corpus; queries about private or other repositories; a defect title that names no surface. Seven more engine queries come from a hunt for parent-class questions across all 300: six by reading, one by a script that follows each named class's `extends` chain.

**Candidate answer files** were located by symbol definition and by reading (namespace to path, then where the named member is defined), by a helper without session context that used no Knowledge Base, memory or GitHub tool. I script-checked every row (the file exists, the symbol stands at the cited line, each class claim re-measured) and read the ten reference answers against their cited lines. Nothing comes from the output of any of the three arms. Trees read: `neomjs/neo` with `dev` at `113f4ef9a8`, this repository at `d6469d3b`.

**Fields.**

- `class`: `parent-class` (the query names a class, and the answering member is defined in an ancestor's file; `chain` lists the files) · `source-only` (in source, and the key symbol has no match under that repository's `learn/`; `guideGrep`) · `source-with-guide-mention` (in source; `learn/` mentions the symbol and does not answer) · `guide-covered` (a file under `learn/` answers and is listed).
- `style`: `question` or `keywords`. The guide bonus counts the question's words, so the two are reported apart.
- `judged` with `referenceAnswer`: the ten rows for the `ask` half.

**One finding before any arm runs.** The log holds almost no parent-class questions. Three of 300 queries name a class whose answering member lives only in an ancestor (E220, E272, E277). In two more the parent holds part of the answer (E242, E282; their `note` says which part). The ticket asks for five, and these are the five. If the verifier does not count the two partial rows, the table reports the parent-class rule on three questions and says so. Writing questions to fill the quota is the trap the ticket names.

**For the verifier.**

1. Each `answerFiles` entry: does that file answer the question? Drop or correct an entry; never add one from an arm's result.
2. Each class claim: the chain, the `learn/` grep.
3. Balance, fixed now: keep every surviving Brain row and the same number of engine rows, all `parent-class` and `source-only` rows first, then the rest in list order.
4. Fix the grep arm's query words and its ordering before the first run.

**Limits to state beside the table.**

- Agents who know this ranking wrote the questions, and 23 of the 49 are keyword queries that name symbols, which favours grep.
- History questions are out. The ticket and pull-request penalties are measured only through the distractors they keep down.
- B076 is the June case the post tells. Flag it or drop it.
- Three rows name an answer file in their own text (E025, E032, B051).

<details><summary>Not carried: 6 of the 55 picked queries</summary>

- `E000` worker.mixin.RemoteMethodAccess: accessing new main threads too early: candidate answer files found, but none that settles the question
- `E141` OffscreenCanvas Canvas worker remote windowId Neo.setupClass: candidate answer files found, but none that settles the question
- `B069` Where is Neo Fleet's Codex Desktop restart and same-thread resume behavior implemented? Distinguish seat resta: candidate answer files found, but none that settles the question
- `B081` Which GraphService, SQLite storage, restore and migration entry points own physical graph transactions and cac: candidate answer files found, but none that settles the question
- `B102` Where does the Fleet Antigravity adapter own or preserve native agent command permissions and terminal auto-ex: candidate answer files found, but none that settles the question
- `B104` Where do Fleet agent settings declare and materialize per-agent harness command approval rules, including Code: no file in the checkout declares it: No file in neo-agent-brain declares or materializes per-agent Codex execpolicy prefix_rule or Claude allowedTools approval rules; only Kimi's [[permis

</details>

<details><summary>fixture.json (49 rows)</summary>

```json
[
 {
  "id": "E002",
  "repo": "neomjs/neo",
  "question": "What is a state provider in Neo?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/datahandling/StateProviders.md",
    "symbol": "Shared Bindable Data (State Providers)",
    "line": 1
   },
   {
    "path": "learn/gettingstarted/ComponentModels.md",
    "symbol": "Shared Bindable Data (a state provider holds properties bound to component configs)",
    "line": 1
   }
  ],
  "asked": 7,
  "lastAsked": "2026-08-11",
  "clusterId": "0a602a46bf84",
  "pick": "list-order"
 },
 {
  "id": "E003",
  "repo": "neomjs/neo",
  "question": "When extending a Neo class, can a subclass redeclare an inherited reactive config with the trailing underscore to change its default value?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/fundamentals/ExtendingNeoClasses.md",
    "symbol": "Overriding an inherited reactive config",
    "line": 57
   },
   {
    "path": "learn/guides/coreengine/SetupClass.md",
    "symbol": "Crucial Rule: trailing underscore exactly once in the prototype chain",
    "line": 50
   },
   {
    "path": "src/Neo.mjs",
    "symbol": "createConfig throws when the config is already reactive on a parent class",
    "line": 355
   }
  ],
  "asked": 4,
  "lastAsked": "2026-08-31",
  "clusterId": "bb42353f9a70",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "No. A reactive config takes its trailing underscore exactly once in the prototype chain; a subclass changes the default with the bare name (e.g. width: 300) and stays reactive, while repeating the underscore makes Neo.setupClass throw."
 },
 {
  "id": "E008",
  "repo": "neomjs/neo",
  "question": "getDomRect windowId Neo.manager.Toast updateItemsInPosition",
  "style": "keywords",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "src/manager/Toast.mjs",
    "symbol": "updateItemsInPosition (fetches DOM rects per windowId realm via getDomRect)",
    "line": 219
   },
   {
    "path": "src/component/Base.mjs",
    "symbol": "getDomRect(id, windowId)",
    "line": 1532
   }
  ],
  "asked": 2,
  "lastAsked": "2026-11-05",
  "clusterId": "68ff5f954805",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "updateItemsInPosition",
   "learnMatches": 0
  }
 },
 {
  "id": "E009",
  "repo": "neomjs/neo",
  "question": "component.wrapper.GoogleMaps markerStore recordChange destroyMarkers onMarkerStoreLoad",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/component/wrapper/GoogleMaps.mjs",
    "symbol": "afterSetMarkerStore (binds load/recordChange to onMarkerStoreLoad/onMarkerRecordChange; onMarkerStoreLoad at 300 calls destroyMarkers)",
    "line": 158
   },
   {
    "path": "src/main/addon/GoogleMaps.mjs",
    "symbol": "destroyMarkers",
    "line": 161
   }
  ],
  "asked": 2,
  "lastAsked": "2026-11-03",
  "clusterId": "c54bb05129a0",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "markerStore",
   "learnMatches": 1
  }
 },
 {
  "id": "E021",
  "repo": "neomjs/neo",
  "question": "In current Neo.mjs DockLayouts, which classes own revealed rail state, projection and in-place retainTopology reconciliation? Does manager or transaction infrastructure know about rails?",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/dashboard/dock/interaction/Rail.mjs",
    "symbol": "class Rail (edge-rail projection, in-place reconcileTabs, reveal state delegated to RevealStateMachine)",
    "line": 51
   },
   {
    "path": "src/dashboard/dock/projection/Reconciler.mjs",
    "symbol": "reconcileProjection (retainTopology in-place item reconciliation)",
    "line": 463
   },
   {
    "path": "src/manager/Transaction.mjs",
    "symbol": "class Transaction (knows no dock, no document, no app class)",
    "line": 46
   }
  ],
  "asked": 2,
  "lastAsked": "2026-09-04",
  "clusterId": "8e0cb1f3c941",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "retainTopology",
   "learnMatches": 1
  }
 },
 {
  "id": "E022",
  "repo": "neomjs/neo",
  "question": "How does grid.Row propagate lifecycle state such as mounted, theme, and windowId to pooled component and canvas cells? Identify owning source and sibling precedents.",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "src/grid/Row.mjs",
    "symbol": "forwardToCells (called from afterSetMounted/afterSetTheme/afterSetWindowId)",
    "line": 142
   },
   {
    "path": "src/table/Body.mjs",
    "symbol": "afterSetWindowId (sibling precedent: relays to createdCellComponents)",
    "line": 181
   },
   {
    "path": "src/container/Base.mjs",
    "symbol": "afterSetWindowId (the item cascade Row replicates by hand)",
    "line": 324
   }
  ],
  "asked": 2,
  "lastAsked": "2026-09-04",
  "clusterId": "5ce92bb35417",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "forwardToCells",
   "learnMatches": 0
  },
  "judged": true,
  "referenceAnswer": "Row.afterSetMounted/afterSetTheme/afterSetWindowId call super, then forwardToCells(key, value) assigns the value to every cell component in this.components: the cascade container.Base does for items, done by hand because a Row is a component. table.Body does the same."
 },
 {
  "id": "E025",
  "repo": "neomjs/neo",
  "question": "Trace src/remotes/Api.mjs loading runtime JSON configuration and deciding data worker execution. Include exact methods and config keys.",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/remotes/Api.mjs",
    "symbol": "load (and register / initAsync)",
    "line": 116
   },
   {
    "path": "src/DefaultConfig.mjs",
    "symbol": "remotesApiUrl",
    "line": 209
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-31",
  "clusterId": "956fec3a12f9",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "workerExecution",
   "learnMatches": 1
  }
 },
 {
  "id": "E032",
  "repo": "neomjs/neo",
  "question": "0029-docking-design DockZoneModel.md DockLayouts guide docking design record",
  "style": "keywords",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/agentos/decisions/0029-docking-design.md",
    "symbol": "ADR 0029: Docking Design",
    "line": 1
   },
   {
    "path": "learn/agentos/DockZoneModel.md",
    "symbol": "Dock-Zone Model Contract",
    "line": 1
   },
   {
    "path": "learn/guides/uibuildingblocks/DockLayouts.md",
    "symbol": "Dock Layouts: One Application, Many Windows",
    "line": 5
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-22",
  "clusterId": "393c050ec7b6",
  "pick": "list-order"
 },
 {
  "id": "E033",
  "repo": "neomjs/neo",
  "question": "Current Neo pattern for button menu config, menu.List createItemContent rich records, keyboard Escape arrows Enter focus-leave dismissal, floating alignment, and Store binding.",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/menu/List.mjs",
    "symbol": "createItemContent (plus onKeyDownEscape 813, onFocusLeave 654, menuFocus_ 69, store config)",
    "line": 427
   },
   {
    "path": "src/button/Base.mjs",
    "symbol": "afterSetMenu (menu_ config creates a floating menu.List aligned t0-b0 to the button)",
    "line": 344
   },
   {
    "path": "src/menu/Model.mjs",
    "symbol": "class Model (menu record fields: iconCls, items, separator, disabled, handler)",
    "line": 7
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-21",
  "clusterId": "e498d0e7e48f",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "menu.List",
   "learnMatches": 1
  }
 },
 {
  "id": "E034",
  "repo": "neomjs/neo",
  "question": "What are Neo's required idioms for data-carrying app UI using data.Store/data.Model and state.Provider, especially navigation lists and shared view state? Cite src contracts and app precedents.",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/datahandling/UsingStores.md",
    "symbol": "Give two views one data owner",
    "line": 70
   },
   {
    "path": "learn/guides/datahandling/StateProviders.md",
    "symbol": "Managing Stores with State Providers",
    "line": 540
   },
   {
    "path": "apps/webdesktop/README.md",
    "symbol": "The pattern (state as permissions, navigation as one TreeStore, views bound to it)",
    "line": 16
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-21",
  "clusterId": "2230673e4109",
  "pick": "list-order"
 },
 {
  "id": "E040",
  "repo": "neomjs/neo",
  "question": "What is the difference between a state.Provider and a data.Store in Neo, and when should a component bind to each?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/datahandling/StateProviders.md",
    "symbol": "Managing Stores with State Providers",
    "line": 540
   },
   {
    "path": "learn/guides/datahandling/Tables.md",
    "symbol": "Put the Store behind a view",
    "line": 143
   },
   {
    "path": "learn/guides/datahandling/UsingStores.md",
    "symbol": "Give two views one data owner",
    "line": 70
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-11",
  "clusterId": "6d84fd929811",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "A state.Provider holds shared bindable view state (data, formulas, stores) that components read hierarchically via bind: {prop: data => ...}; a data.Store owns the record population. Put the Store in the view-root provider and bind components to it with bind: {store: 'stores.key'}."
 },
 {
  "id": "E050",
  "repo": "neomjs/neo",
  "question": "state provider hierarchical data binding",
  "style": "keywords",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/datahandling/StateProviders.md",
    "symbol": "The Hierarchical Data Proxy",
    "line": 25
   },
   {
    "path": "src/state/createHierarchicalDataProxy.mjs",
    "symbol": "createHierarchicalDataProxy",
    "line": 136
   },
   {
    "path": "learn/blog/v10-deep-dive-state-provider.md",
    "symbol": "Hierarchical by Design: Nested Providers That Just Work",
    "line": 223
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-07",
  "clusterId": "ef1d34b5678d",
  "pick": "list-order"
 },
 {
  "id": "E052",
  "repo": "neomjs/neo",
  "question": "grid buffered row rendering performance",
  "style": "keywords",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/datahandling/Grids.md",
    "symbol": "Optimizing Virtual Rendering (bufferRowRange / bufferColumnRange)",
    "line": 955
   },
   {
    "path": "src/grid/Body.mjs",
    "symbol": "bufferRowRange_",
    "line": 97
   },
   {
    "path": "learn/guides/datahandling/DynamicGrids.md",
    "symbol": "Vertical Virtualization (Row Pooling)",
    "line": 135
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-04",
  "clusterId": "e04729df2dfc",
  "pick": "list-order"
 },
 {
  "id": "E061",
  "repo": "neomjs/neo",
  "question": "In which thread does a component's code run in Neo.mjs, and how does a config change on a component reach the real DOM?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/guides/fundamentals/WorkerArchitecture.md",
    "symbol": "The Triangular Communication (VDOM Updates)",
    "line": 130
   },
   {
    "path": "learn/benefits/ArchitectureOverview.md",
    "symbol": "Key Concepts (App Worker hosts components; VDom Worker diffs; main thread applies deltas)",
    "line": 92
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "6b24ba71f1f3",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "Component code runs in the App Worker. A config change updates the component's VDOM there, which goes to the VDom Worker for diffing; the main thread intercepts the deltas reply, applies them to the DOM via Neo.worker.Manager.handleDomUpdate(), then confirms to the App Worker."
 },
 {
  "id": "E066",
  "repo": "neomjs/neo",
  "question": "Gantt DependencyStore getPredecessors getSuccessors",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/gantt/DependencyStore.mjs",
    "symbol": "getPredecessors (getSuccessors at 121)",
    "line": 96
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "7f518e3a5401",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "DependencyStore",
   "learnMatches": 2
  }
 },
 {
  "id": "E072",
  "repo": "neomjs/neo",
  "question": "How does application code read the currently selected record of a Neo.grid.Container row selection (selectionModel, selectedRows, useInternalId, getRowRecord, selectionChange)?",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/selection/grid/BaseModel.mjs",
    "symbol": "getRowRecord (selectedRows_ at 38; fireRowSelectionChange at 365)",
    "line": 428
   },
   {
    "path": "src/grid/View.mjs",
    "symbol": "get selectedRows (selectionModel_ config at 132)",
    "line": 351
   },
   {
    "path": "src/grid/Container.mjs",
    "symbol": "useInternalId_",
    "line": 241
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "f2a633af538f",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "selectionModel",
   "learnMatches": 2
  }
 },
 {
  "id": "E079",
  "repo": "neomjs/neo",
  "question": "How does TreeStore ancestor-aware filtering keep nodes, and how should Menu.List consume the filtered view while getChildren remains structural?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "src/data/TreeStore.mjs",
    "symbol": "filter (ancestor-aware _keptNodes); isKept at 792; getChildren at 767",
    "line": 610
   },
   {
    "path": "src/menu/List.mjs",
    "symbol": "syncLevelRecords (getChildren + isKept)",
    "line": 1008
   },
   {
    "path": "learn/guides/datahandling/TreeStore.md",
    "symbol": "Ancestor-Aware Filtering",
    "line": 323
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "8b30a877f344",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "TreeStore.filter() keeps a node that matches, whose ancestor matched, or whose descendant matched (auto-expanding ancestors); getChildren() reads the Structural Layer and still returns filtered-out children, so Menu.List.syncLevelRecords() filters them with TreeStore#isKept()."
 },
 {
  "id": "E092",
  "repo": "neomjs/neo",
  "question": "How do Store.updateRows, Store.writeRows and TreeStore Structural Layer versus visible projection handle raw Turbo Mode row batches, nested fields, hydration and delta publication?",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/data/Store.mjs",
    "symbol": "updateRows (writeRows at 1902)",
    "line": 1837
   },
   {
    "path": "src/data/TreeStore.mjs",
    "symbol": "getRow (answers from the Structural Layer)",
    "line": 814
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "1f96fced0b32",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "Turbo Mode",
   "learnMatches": 4
  }
 },
 {
  "id": "E107",
  "repo": "neomjs/neo",
  "question": "Neo.core.Config initDescriptor stale JSDoc cloneOnGet merge default replace",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/core/Config.mjs",
    "symbol": "initDescriptor (JSDoc for clone, cloneOnGet, isEqual, merge)",
    "line": 99
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "bfd3dbb475e5",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "cloneOnGet",
   "learnMatches": 1
  }
 },
 {
  "id": "E120",
  "repo": "neomjs/neo",
  "question": "Neo.data.connection.Xhr promiseJson onLoad HTTP error status Store load Pipeline",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/data/connection/Xhr.mjs",
    "symbol": "onLoad (success only for 2xx) and promiseJson (rejects refused answers)",
    "line": 95
   },
   {
    "path": "src/data/Store.mjs",
    "symbol": "#loadRecords (pipeline.read; failures rethrown)",
    "line": 1227
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "5e6c7db30ebe",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "connection.Xhr",
   "learnMatches": 1
  }
 },
 {
  "id": "E126",
  "repo": "neomjs/neo",
  "question": "Neo data Model nested fields and collection Filter fields dependency declaration compared with property paths and selective reapplyOnRecordChange",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "src/data/Store.mjs",
    "symbol": "#reads (filter/sorter fields vs changed dotted paths; shouldReapplyFilters at 1714)",
    "line": 1752
   },
   {
    "path": "src/collection/Filter.mjs",
    "symbol": "fields_ (dependency declaration; reapplyOnRecordChange at 99)",
    "line": 51
   },
   {
    "path": "src/data/Model.mjs",
    "symbol": "hasNestedFields",
    "line": 99
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "b5bd6751c7fe",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "reapplyOnRecordChange",
   "learnMatches": 2
  }
 },
 {
  "id": "E132",
  "repo": "neomjs/neo",
  "question": "main addon Clipboard paste cut fulfillWrite DOM event payload",
  "style": "keywords",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "src/main/addon/Clipboard.mjs",
    "symbol": "fulfillWrite (onCopyOrCut at 426, onPaste at 513)",
    "line": 269
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "65aae06d2c9a",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "fulfillWrite",
   "learnMatches": 0
  }
 },
 {
  "id": "E183",
  "repo": "neomjs/neo",
  "question": "Store TreeStore getCanonicalKey getKey hydrateRecord key type String numeric id",
  "style": "keywords",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "src/data/Store.mjs",
    "symbol": "getCanonicalKey (getKey at 691)",
    "line": 1597
   },
   {
    "path": "src/data/TreeStore.mjs",
    "symbol": "get (falls back to getCanonicalKey; hydrateRecord at 923)",
    "line": 720
   },
   {
    "path": "learn/guides/datahandling/DataPipelines.md",
    "symbol": "The pushed key is canonicalized before lookup",
    "line": 225
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-14",
  "clusterId": "f9b261d12d0a",
  "pick": "parent-class-hunt",
  "note": "the query names Store and TreeStore, and Store.mjs defines the member: not a parent-class case"
 },
 {
  "id": "E220",
  "repo": "neomjs/neo",
  "question": "Neo.button.Base ui config values (primary, secondary, tertiary, ghost) and pressed toggle state; is there a segmented button or button group for mutually exclusive options in a toolbar?",
  "style": "question",
  "class": "parent-class",
  "answerFiles": [
   {
    "path": "src/component/Base.mjs",
    "symbol": "ui_ ('primary' and other attributes)",
    "line": 323
   },
   {
    "path": "src/button/Base.mjs",
    "symbol": "pressed_",
    "line": 129
   },
   {
    "path": "learn/guides/uibuildingblocks/StylingAndTheming.md",
    "symbol": "ui",
    "line": 67
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-13",
  "clusterId": "8029366c843d",
  "pick": "parent-class-hunt",
  "named": "src/button/Base.mjs",
  "chain": [
   "src/button/Base.mjs",
   "src/component/Base.mjs"
  ]
 },
 {
  "id": "E224",
  "repo": "neomjs/neo",
  "question": "How does the calendar week view (and day view) place event records, and is there any row or handling for all-day or multi-day events (the Event model's allDay field)?",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "src/calendar/view/week/Component.mjs",
    "symbol": "updateEvents",
    "line": 910
   },
   {
    "path": "src/calendar/store/Events.mjs",
    "symbol": "getDayRecords",
    "line": 40
   },
   {
    "path": "src/calendar/model/Event.mjs",
    "symbol": "allDay field",
    "line": 22
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-13",
  "clusterId": "0c9d9ac8e08a",
  "pick": "parent-class-hunt",
  "guideGrep": {
   "pattern": "allDay",
   "learnMatches": 0
  }
 },
 {
  "id": "E242",
  "repo": "neomjs/neo",
  "question": "How does a flat filtered Neo Store maintain order and identity of visible key and internalId Maps across filter or sort, including callers retaining .map? How is a population row's insertion position represented?",
  "style": "question",
  "class": "parent-class",
  "answerFiles": [
   {
    "path": "src/collection/Base.mjs",
    "symbol": "filter (map/internalIdMap cleared and refilled in place; initialIndexCounter assigned in splice at 1650)",
    "line": 834
   },
   {
    "path": "src/data/Store.mjs",
    "symbol": "sort (restores order via Symbol.for('initialIndex'))",
    "line": 2224
   },
   {
    "path": "src/data/RecordFactory.mjs",
    "symbol": "Record [initialIndexSymbol] = null",
    "line": 193
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-13",
  "clusterId": "f971889481d5",
  "pick": "parent-class-hunt",
  "named": "src/data/Store.mjs",
  "chain": [
   "src/data/Store.mjs",
   "src/collection/Base.mjs"
  ],
  "note": "partial: Store.mjs:783 overrides filter() for Turbo Mode and delegates; the map handling the question asks about is the parent body"
 },
 {
  "id": "E272",
  "repo": "neomjs/neo",
  "question": "Tooltip show duplicate DOM nodes pending initVnode hide mountFlight startMountFlight issue 19265 19268",
  "style": "keywords",
  "class": "parent-class",
  "answerFiles": [
   {
    "path": "src/component/Base.mjs",
    "symbol": "startMountFlight (mountFlight latch used by show/hide)",
    "line": 2014
   },
   {
    "path": "src/tooltip/Base.mjs",
    "symbol": "show (single mount flight; hide records intent at 278)",
    "line": 380
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-12",
  "clusterId": "ac7ea7153afd",
  "pick": "parent-class-hunt",
  "named": "src/tooltip/Base.mjs",
  "chain": [
   "src/tooltip/Base.mjs",
   "src/container/Base.mjs",
   "src/component/Base.mjs"
  ]
 },
 {
  "id": "E277",
  "repo": "neomjs/neo",
  "question": "Neo.draggable.container.SortZone onDragEnd processDragEnd queueDragStep lifecycle async move cancellation",
  "style": "keywords",
  "class": "parent-class",
  "answerFiles": [
   {
    "path": "src/draggable/container/DragZone.mjs",
    "symbol": "queueDragStep",
    "line": 162
   },
   {
    "path": "src/draggable/container/SortZone.mjs",
    "symbol": "onDragEnd (processDragEnd at 656)",
    "line": 588
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-12",
  "clusterId": "5262eea71fc6",
  "pick": "parent-class-hunt",
  "named": "src/draggable/container/SortZone.mjs",
  "chain": [
   "src/draggable/container/SortZone.mjs",
   "src/draggable/container/DragZone.mjs"
  ],
  "note": "found by the mechanical hunt (hunt.py): queueDragStep is defined in the parent only"
 },
 {
  "id": "E282",
  "repo": "neomjs/neo",
  "question": "Neo.data.Store local filter event allItems get(key) canonical record identity and listener teardown",
  "style": "keywords",
  "class": "parent-class",
  "answerFiles": [
   {
    "path": "src/collection/Base.mjs",
    "symbol": "allItems (config; filter at 834 fires 'filter')",
    "line": 49
   },
   {
    "path": "src/data/Store.mjs",
    "symbol": "get (the key lookup Store overrides; onAllItemsRecordChange at 1698)",
    "line": 881
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-12",
  "clusterId": "2241b9079366",
  "pick": "parent-class-hunt",
  "named": "src/data/Store.mjs",
  "chain": [
   "src/data/Store.mjs",
   "src/collection/Base.mjs"
  ],
  "note": "partial: allItems and the filter event are the parent's, get(key) and the listener teardown are Store's own"
 },
 {
  "id": "B004",
  "repo": "neomjs/neo-agent-brain",
  "question": "Tenant ingest builds its temp JSONL as one string and hits V8's limit",
  "style": "keywords",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/knowledge-base/IngestionService.mjs",
    "symbol": "writeTempJsonl (streams one record at a time to stay under V8's string cap)",
    "line": 3130
   }
  ],
  "asked": 3,
  "lastAsked": "2026-11-11",
  "clusterId": "136cb43aeef0",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "writeTempJsonl",
   "learnMatches": 0
  }
 },
 {
  "id": "B005",
  "repo": "neomjs/neo-agent-brain",
  "question": "Shape B a2a webhook stale unread wake freshness CoalescingEngine #15704",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/wakeCoalescePolicy.mjs",
    "symbol": "isMessageWakeFresh (partitionMessageWakesByFreshness at 120)",
    "line": 84
   },
   {
    "path": "ai/services/memory-core/CoalescingEngineService.mjs",
    "symbol": "_partitionExpiredMessageWakes (applied at _flush 634)",
    "line": 801
   }
  ],
  "asked": 3,
  "lastAsked": "2026-08-11",
  "clusterId": "3d00966d0364",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "CoalescingEngine",
   "learnMatches": 1
  }
 },
 {
  "id": "B015",
  "repo": "neomjs/neo-agent-brain",
  "question": "Brain #809 startAgentProvisioned resolveSeatPlaneTarget FleetTenantService storeSeatPlaneCredential existing PAT tenant branch start tests unit spec",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/fleet/startAgentProvisioned.mjs",
    "symbol": "provisionAgent plane branch (storeSeatPlaneCredential ifAbsent for the existing PAT; tenant branch at 545)",
    "line": 585
   },
   {
    "path": "ai/services/fleet/resolveSeatPlaneTarget.mjs",
    "symbol": "resolveSeatPlaneTarget",
    "line": 27
   },
   {
    "path": "ai/services/fleet/FleetTenantService.mjs",
    "symbol": "storeSeatPlaneCredential",
    "line": 375
   }
  ],
  "asked": 2,
  "lastAsked": "2026-10-03",
  "clusterId": "2ef80ff2f108",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "resolveSeatPlaneTarget",
   "learnMatches": 1
  }
 },
 {
  "id": "B023",
  "repo": "neomjs/neo-agent-brain",
  "question": "What source-of-authority documents and current code define Agent OS seat-hook projection, generated-not-tracked custody, and the Claude settings reconciliation boundary?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/agentos/decisions/0040-agentos-extraction-topology.md",
    "symbol": "§2.7 Wave-one custody (seat hooks as generated-not-tracked artifacts; §2.5 at 136)",
    "line": 205
   },
   {
    "path": "ai/scripts/lifecycle/hooks/projectSeatHooks.mjs",
    "symbol": "reconcileClaudeSettings (module projects seat hooks)",
    "line": 1071
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-31",
  "clusterId": "17b5c625218d",
  "pick": "list-order"
 },
 {
  "id": "B042",
  "repo": "neomjs/neo-agent-brain",
  "question": "add_memory post-WAL synchronous mailbox delta response timeout accepted write stage timings",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/MemoryService.mjs",
    "symbol": "addMemory (stageTimings, mailbox omitted, POST_WAL_RESPONSE_BUDGET_MS at 200)",
    "line": 629
   },
   {
    "path": "ai/mcp/server/memory-core/openapi.yaml",
    "symbol": "stageTimings response schema",
    "line": 5714
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-10",
  "clusterId": "6afdd105a648",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "add_memory",
   "learnMatches": 10
  }
 },
 {
  "id": "B043",
  "repo": "neomjs/neo-agent-brain",
  "question": "Ollama early disconnect leaves embedding runner CPU bound after client and sockets disappear",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/TextEmbeddingService.mjs",
    "symbol": "settleCallerWhileProviderContinues (native Ollama keeps computing after the client socket closes)",
    "line": 431
   },
   {
    "path": "ai/services/shared/providerActivityLedger.mjs",
    "symbol": "reapProviderActivity",
    "line": 415
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-10",
  "clusterId": "7ffeb31279b3",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "native-Ollama",
   "learnMatches": 1
  }
 },
 {
  "id": "B048",
  "repo": "neomjs/neo-agent-brain",
  "question": "How do blobless tenant repository mirrors authenticate lazy git show content reads? Identify which functions own credentialRef and knownHostsPath, the envelope-builder carrier, and which Git operations should remain credential-free.",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/knowledge-base/helpers/gitMirror.mjs",
    "symbol": "readRevisionBlob (runGit at 480, getKnownHostsPath at 812)",
    "line": 1444
   },
   {
    "path": "ai/services/knowledge-base/helpers/tenantRepoIngestEnvelopeBuilder.mjs",
    "symbol": "buildIngestEnvelope (credentialRef carrier)",
    "line": 381
   },
   {
    "path": "ai/services/knowledge-base/helpers/repositoryRevisionReader.mjs",
    "symbol": "createRepositoryRevisionReader (passes credentialRef to readRevisionBlob)",
    "line": 133
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-07",
  "clusterId": "7bce4a01d36b",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "credentialRef",
   "learnMatches": 6
  }
 },
 {
  "id": "B049",
  "repo": "neomjs/neo-agent-brain",
  "question": "What contracts govern TenantRepoSyncService scheduling, consecutiveFailures, lastRunAttemptAt, persisted revisions, leases, and crash recovery in the Neo orchestrator?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/agentos/cloud-deployment/TenantIngestionModel.md",
    "symbol": "Mirror Volume (revisions manifest, lease, atomic writes)",
    "line": 317
   },
   {
    "path": "ai/daemons/orchestrator/services/TenantRepoSyncService.mjs",
    "symbol": "class TenantRepoSyncService",
    "line": 887
   },
   {
    "path": "ai/daemons/orchestrator/services/tenantRepoCheckpointValidity.mjs",
    "symbol": "normalizeTenantRepoCheckpointState",
    "line": 97
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-07",
  "clusterId": "e7af1beffb1e",
  "pick": "list-order"
 },
 {
  "id": "B051",
  "repo": "neomjs/neo-agent-brain",
  "question": "What are the authoritative backup bundle retention and recoverability semantics in ai/scripts/maintenance/backup.mjs, especially keepMinimum, bundle-meta.json, empty per-substrate payloads, safe failure behavior, and restore selection?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "ai/scripts/maintenance/backup.mjs",
    "symbol": "cleanOldBackups (classifyBundleRecoverability at 1147)",
    "line": 1323
   },
   {
    "path": "ai/scripts/maintenance/restore.mjs",
    "symbol": "verifyLatestBackupRestorable",
    "line": 968
   },
   {
    "path": "learn/agentos/tooling/RestorationRunbook.md",
    "symbol": "Reading `bundle-meta.json`: two blocks, two different questions",
    "line": 128
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-07",
  "clusterId": "54fe6f3046ee",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "cleanOldBackups keeps the newest keepMinimum restorable bundles regardless of age; the floor counts recoverability, not directories, and a bundle survives if either keepMinimum or maxDays protects it."
 },
 {
  "id": "B054",
  "repo": "neomjs/neo-agent-brain",
  "question": "What contracts and prior decisions govern MemoryService buildMiniSummary failure classification, empty or whitespace model output, timeout boundaries, and cause observability?",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/MemoryService.mjs",
    "symbol": "buildMiniSummary (backfillMiniSummaries at 2485)",
    "line": 2078
   },
   {
    "path": "ai/configBase.mjs",
    "symbol": "memoryService leaves (generateMiniSummaryTimeoutMs, miniSummaryTimeoutMs)",
    "line": 1338
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-02",
  "clusterId": "46d432287439",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "buildMiniSummary",
   "learnMatches": 0
  },
  "judged": true,
  "referenceAnswer": "buildMiniSummary never throws and returns {summary, cause}: whitespace-only output normalizes to '' and is empty-output; no provider is no-model; wrapper or provider timeouts are timeout-inner (with timeoutCode); other errors are provider-error; its budget is generateMiniSummaryTimeoutMs."
 },
 {
  "id": "B056",
  "repo": "neomjs/neo-agent-brain",
  "question": "In Neo's RuntimeFreshnessService, what is the authority boundary between observed fieldKeys, statusFields, identityLabel wording, and the final current/stale verdict? Cite the owning source and tests.",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/mcp/server/shared/services/RuntimeFreshnessService.mjs",
    "symbol": "FRESHNESS_DIMENSIONS (resolveStatusFields at 70, assertLabelIsBacked at 123)",
    "line": 106
   }
  ],
  "asked": 2,
  "lastAsked": "2026-08-01",
  "clusterId": "0840a31a2a92",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "RuntimeFreshnessService",
   "learnMatches": 0
  },
  "judged": true,
  "referenceAnswer": "A label may claim source, config or schema only if FRESHNESS_DIMENSIONS finds a status-driving field for it (gitHead for source, a config key, an openapi or schema key); otherwise assertLabelIsBacked throws when the tracker is constructed."
 },
 {
  "id": "B073",
  "repo": "neomjs/neo-agent-brain",
  "question": "Where does the Fleet open-work producer normalize PR comments, retain review witnesses, and acquire failed-check evidence for repair-ci wakes?",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/fleet/openWorkReducer.mjs",
    "symbol": "normalizePullRequest",
    "line": 137
   },
   {
    "path": "ai/services/fleet/openWorkReviewRecords.mjs",
    "symbol": "retainReviewWitnesses",
    "line": 105
   },
   {
    "path": "ai/services/fleet/wireFleetOpenWorkWakes.mjs",
    "symbol": "failedChecksReader (failedChecksOf at 108)",
    "line": 516
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "fc4997f82c86",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "retainReviewWitnesses",
   "learnMatches": 0
  }
 },
 {
  "id": "B076",
  "repo": "neomjs/neo-agent-brain",
  "question": "Which embedding provider does the Knowledge Base use, and is a Gemini API key required to run it?",
  "style": "question",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "learn/agentos/KnowledgeBase.md",
    "symbol": "The Runtime Shape (embedding generation is provider-aware)",
    "line": 100
   },
   {
    "path": "learn/agentos/ModelProviders.md",
    "symbol": "Configure the contract, not the brand",
    "line": 127
   },
   {
    "path": "ai/configBase.mjs",
    "symbol": "embeddingProvider leaf (default openAiCompatible)",
    "line": 971
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-16",
  "clusterId": "ab7576ebf6c7",
  "pick": "list-order",
  "judged": true,
  "referenceAnswer": "Embeddings use embeddingProvider (NEO_EMBEDDING_PROVIDER), defaulting to the local openAiCompatible route; ollama is also local, and only the gemini provider needs GEMINI_API_KEY, so no Gemini key is required by default."
 },
 {
  "id": "B086",
  "repo": "neomjs/neo-agent-brain",
  "question": "Where are current PR commit identities and per-commit diff evidence fetched and passed to resolveCrossFamilyVerdict and openWorkHolder? Identify the actual merge-readiness and Fleet snapshot consumers.",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/github-workflow/PullRequestService.mjs",
    "symbol": "resolveCrossFamilyVerdict call in merge readiness (authorDisclosedFamilies at 599)",
    "line": 1000
   },
   {
    "path": "ai/services/fleet/openWorkReducer.mjs",
    "symbol": "normalizePullRequest (disclosedFamilies from commitMessages)",
    "line": 137
   },
   {
    "path": "ai/services/fleet/fleetOpenWorkSource.mjs",
    "symbol": "holderOf consumer of the Fleet snapshot",
    "line": 35
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "2672fe2377cc",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "resolveCrossFamilyVerdict",
   "learnMatches": 0
  }
 },
 {
  "id": "B088",
  "repo": "neomjs/neo-agent-brain",
  "question": "Where does update_issue_relationship resolve child_issue and parent_issue IDs, and does it support different repositories for the two issue numbers?",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/github-workflow/IssueService.mjs",
    "symbol": "updateIssueRelationship",
    "line": 2122
   },
   {
    "path": "ai/mcp/server/github-workflow/openapi.yaml",
    "symbol": "update_issue_relationship",
    "line": 2038
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "862ee2ee488f",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "update_issue_relationship",
   "learnMatches": 2
  },
  "judged": true,
  "referenceAnswer": "IssueService.updateIssueRelationship resolves both issue numbers inside one repository target and refuses repository-qualified inputs with CROSS_REPOSITORY_RELATIONSHIP_UNSUPPORTED, so the two issues cannot be in different repositories."
 },
 {
  "id": "B112",
  "repo": "neomjs/neo-agent-brain",
  "question": "CoalescingEngineService stagePrimaryWake high priority wake refractory coalescing",
  "style": "keywords",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/CoalescingEngineService.mjs",
    "symbol": "class CoalescingEngineService (urgent window and refractory; _isUrgentQueuedEvent at 501)",
    "line": 104
   },
   {
    "path": "ai/daemons/wake/primaryWakeSpool.mjs",
    "symbol": "stagePrimaryWake",
    "line": 114
   },
   {
    "path": "ai/services/memory-core/wakeCoalescePolicy.mjs",
    "symbol": "isUrgentWakeEvent (computeUrgentFlushHoldMs at 319)",
    "line": 238
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "19bb7adc14c6",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "CoalescingEngineService",
   "learnMatches": 1
  }
 },
 {
  "id": "B116",
  "repo": "neomjs/neo-agent-brain",
  "question": "IssueIngestor normalizeLegacyAnchoredIssueEdgesOnce legacy anchor weight reset 1527 1729",
  "style": "keywords",
  "class": "guide-covered",
  "answerFiles": [
   {
    "path": "ai/scripts/migrations/normalizeLegacyIssueEdgeWeights.mjs",
    "symbol": "normalizeLegacyIssueEdgeWeights (the retired inline reset, now an upgrade CLI)",
    "line": 202
   },
   {
    "path": "learn/agentos/DeploymentCookbook.md",
    "symbol": "Persistent graph-store upgrade: legacy issue-edge weights",
    "line": 195
   },
   {
    "path": "ai/services/ingestion/IssueIngestor.mjs",
    "symbol": "ingestIssueStates",
    "line": 289
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "556516d11ced",
  "pick": "list-order"
 },
 {
  "id": "B117",
  "repo": "neomjs/neo-agent-brain",
  "question": "promoteStandaloneWal memoryWalStore standalone promote",
  "style": "keywords",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/memory-core/helpers/promoteStandaloneWal.mjs",
    "symbol": "promoteStandaloneWal",
    "line": 334
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "b20440cbd849",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "promoteStandaloneWal",
   "learnMatches": 0
  }
 },
 {
  "id": "B135",
  "repo": "neomjs/neo-agent-brain",
  "question": "Where does SourceParser inspectSourceModule provide class and method extraction for the shared source-module graph?",
  "style": "question",
  "class": "source-with-guide-mention",
  "answerFiles": [
   {
    "path": "ai/services/knowledge-base/helpers/inspectSourceModule.mjs",
    "symbol": "inspectSourceModule",
    "line": 579
   },
   {
    "path": "ai/services/knowledge-base/parser/SourceParser.mjs",
    "symbol": "parseModuleSource (wraps inspectSourceModule)",
    "line": 14
   },
   {
    "path": "ai/services/knowledge-base/helpers/sourceTreeIndex.mjs",
    "symbol": "inspectSourceModule call with extractCalls",
    "line": 226
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "ef19cd474931",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "SourceParser",
   "learnMatches": 3
  }
 },
 {
  "id": "B138",
  "repo": "neomjs/neo-agent-brain",
  "question": "Which existing Fleet start path provisions supported desktops and arms their private primary session wake route, and how is receiver state directory supplied to a packaged Fleet?",
  "style": "question",
  "class": "source-only",
  "answerFiles": [
   {
    "path": "ai/services/fleet/FleetManager.mjs",
    "symbol": "startAgent",
    "line": 621
   },
   {
    "path": "ai/services/fleet/devFleetServer.mjs",
    "symbol": "wakeStateOptions.wakeReceiverStateDir",
    "line": 425
   },
   {
    "path": "ai/configBase.mjs",
    "symbol": "fleet.wakeReceiverStateDir leaf (NEO_WAKE_RECEIVER_STATE_DIR)",
    "line": 544
   }
  ],
  "asked": 1,
  "lastAsked": "2026-11-15",
  "clusterId": "88c3c792c60e",
  "pick": "list-order",
  "guideGrep": {
   "pattern": "wakeReceiverStateDir",
   "learnMatches": 0
  }
 }
]
```

</details>

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code


- 2026-11-16T13:47:23Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-16T14:31:25Z @neo-gemini-maryam cross-referenced by #1806
- 2026-11-16T14:31:39Z @neo-gemini-maryam cross-referenced by #1807
- 2026-11-16T14:34:35Z @neo-gemini-maryam cross-referenced by PR #1808
- 2026-11-16T16:29:25Z @neo-fable-clio cross-referenced by #419
- 2026-11-16T17:30:43Z @neo-gemini-maryam referenced in commit `8021d81` - "fix(kb): add distinct-word ripgrep arm and direct config leaf reads (#1800)"

