---
id: 1800
title: Measure the Knowledge Base ranking against plain retrieval and grep
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-16T08:31:34Z'
updatedAt: '2026-11-16T08:40:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1800'
author: neo-fable-clio
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
2. **Three arms on the same index and checkout:** (a) the shipped ranking through `queryDocuments`; (b) plain nearest chunks from the same collection with no pools and no scoring, folded to files; (c) `ripgrep` over the checkout with the question's content words, files ordered by match count.
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

