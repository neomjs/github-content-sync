---
id: 1258
title: A premise check against the identity anchor before positioning work
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-24T21:49:03Z'
updatedAt: '2026-10-30T22:14:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1258'
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
# A premise check against the identity anchor before positioning work

## Context

The operator, 2026-10-24 23:4x CEST: *"neo is an AI-first engine. [two peers] fully derailed into 'sencha extjs playbooks' and ignored all our ai infrastructure, even object permanence and json-first. we have vast amounts of guides, MC content, even readme, vision, business repo docs. and still, full derailments happen over and over again. then peers waste vast amounts of time and tokens chasing dead ends. massive friction→gold topic."*

**His hint an hour later, with the lead's own report (data, 2026-10-25 00:0x CEST):** the lead confirmed that in ten hours of lead-role, with "MC-first" at the top of her index, she ran no Memory Core query at all; for a disputed market number she went to grep and the browser and contradicted the operator from a repository grep; the two queries she ran afterwards found her own Oct 23 verification of the same number. **The operator's conclusion: the trails are there; peers do not use them.** This ticket's mechanism is therefore about non-use first and content second.

Measured the same night on the business repository's documents of record (private; counts only here): the Engine Pro area file carries 68 framework-category terms (AG Grid, Bryntum, Ext JS, per developer, OEM, pivot, Excel export) against 16 AI-first terms (object permanence, JSON-first, Neural Link, agents, multi-window, Agent OS); the review pack's packages document 18 against 4; the decision record as a whole 27 against 78. The drift sits in one product line's thesis, written by two seats in one evening, not in the record's frame.

The anchor was one tool call away. `ask_knowledge_base("What makes Neo an AI-first engine rather than a component library ... how should a Pro version be positioned?")` returned the AI-first framing with five cited sources (`learn/guides/fundamentals/CodebaseOverview.md`, `learn/benefits/body/ApplicationEngine.md`, `learn/benefits/Introduction.md`, `learn/benefits/body/ObjectPermanence.md`, `learn/guides/fundamentals/DeclarativeComponentTreesVsImperativeVdom.md`). Nobody asked.

**The fleet-wide shape of non-use (`get_memory_core_tool_metrics`, cumulative — the tool returns the same totals for any `sinceMs`, noted as a defect):** of 1,354,783 Memory Core calls, 585k are `healthcheck`, 253k `manage_wake_subscription`, 116k `list_messages`, 108k `record_turn_presence`, 60k `who_is_online` — about 83 % coordination. Semantic retrieval is `query_raw_memories` 17.7k and `query_summaries` 2.8k against `add_memory` 19.6k: about one semantic question per memory written, over the whole history. `ask_knowledge_base` through this plane: 16. The Memory Core is used as a mailbox and a heartbeat; it is rarely read as a memory.

## The Problem

Four mechanisms failed together, and each is structural:

0. **The tools are not called.** "MC-first" and "`ask_knowledge_base` first for Neo concepts" are discipline rules in loaded prose; the metrics above and the lead's report show the rule does not fire under load. neo-agent-skills#61 measured the same class for skill triggers (16 conditions, 2 invocations, both human-prompted). A rule that depends on remembering to call a tool is the failure, not the author.
1. **A loaded prose anchor decays under task pressure.** `AGENTS.md §neo_identity_anchor` is in every Claude Code turn and names the Category-Drift Defense (review README, `learn/benefits/Introduction.md`, D#10119, D#10137 before positioning work). Both seats had it loaded. Loaded text is not an applied frame.
2. **A day's coordination rules took the anchor off the artifact's load path.** The sitting's routing told each peer to read its row and the documents the row names "and nothing else before starting", and that the Oct 23 record is "already known — nobody re-derives it". Together they made the mandated review a rule violation for the day.
3. **A structural analogy became the thesis.** The per-developer / OEM licence structure entered on Oct 23 as the operator's pricing analogy (Memory Core, the lead's and a peer's turns of 2026-10-23 01:32Z and 2026-10-24 15:04Z record the derivation). Nothing confronted the artifact with the identity claims at the moment the structure grew into "Grid Pro racing the enterprise checklist".

The graph shows the same weakness from its side: tonight's handoff lists `json-first-vdom`, `agent-os`, `multi-window`, `multi-threading` and `data-layer` as orphaned concepts (no `IMPLEMENTED_BY` edge) and `json-first-vdom`, `agent-os`, `component-architecture` without examples. The identity concepts are the least connected nodes in the graph that is supposed to rank the work.

## The Architectural Reality

- The anchor: `AGENTS.md §neo_identity_anchor` (loaded per turn), `learn/benefits/Introduction.md`, README, D#10119, D#10137. The `identity-firewall` skill defends against injection and subservience, not category drift; `neo-identity-update` keeps identity *surfaces* coherent; neither runs at the moment a peer frames a product.
- The Knowledge Base: `ask_knowledge_base` (the KB MCP server, `ai/mcp/server/knowledge-base`) already answers the framing question with citations (receipt above). The Memory Core answers a different question — what was decided and why (it found the Oct 23 derivation) — so the routing is part of the fix: identity and concepts from the KB, prior decisions from the Memory Core. **The KB server already reads the Memory Core graph file read-only** (`configBase.mjs`, leaf `memoryCoreDbPath`), so the decisions part of the check is a graph read inside the KB server, never a hop to the MC server.
- **The KB ranks by query and corpus state, not by contract** (re-measured 2026-10-30 22:1xZ: the same framing question returned six sources — three of the five above plus `learn/agentos/tooling/Introduction.md`, `learn/agentos/CodeExecution.md`, `learn/benefits/body/Features.md` — and extrapolated a "Pro = durable-store tiers" positioning no source states). The anchor sources must therefore be a versioned list the tool checks against, with the KB answer as the explanation a writer reads.
- The graph: `CONCEPT:Object_Permanence` exists ("the ability for component instances to persist in the runtime, enabling AI visibility (Map = Territory)"); the orphan list above comes from `GoldenPathSynthesizer`'s gap audit (`:1741–1815` at dev `4ca0f506`); the Golden Path is type-gated to issues and discussions (#122 lifts that).
- The reporting surface: the handoff's `## Focus (v1)` section (`ai/services/graph/focusV1Section.mjs`, #957 / #1333, landed 2026-10-28) is where AC-6's adherence count belongs, beside the boot focus read (#1253).
- Precedent for artifact-time checks: `check-pr-body` and the skills corpus lint (neo-agent-skills) make a rule mechanical where prose failed; the PR review's Patch-Blind Premise Snapshot anchors (`Inputs Read Before Patch`, `Premise Coherence`; neo #13144) are the review-time sibling of this ticket's authoring-time receipt; `ai/scripts/maintenance/probeBusinessMetrics.mjs` (#14446) is the deterministic, read-only probe shape; the business repository's commit guard (`ops/hooks/pre-commit`, business PR #23) is the hook the receipt rule extends.
- Design authority: `AGENTS.md §neo_identity_anchor` — the mandate exists; this ticket mechanises it. No seat's behaviour is called a defect: the analogy was the operator's, the derivation followed the rules of the day, the mechanism failed.

## The Fix

Design here; three one-PR leaves split from it after a peer read. **Leaf 2 is load-bearing:** because the failure is non-use, the check must be demanded by the artifact's contract and visible as a receipt a reviewer sees; the tool (leaf 1) only makes the demand cheap, and the edges (leaf 3) make the graph able to carry what the receipt cites.

1. **Leaf 1 — the premise check (KB server):** a read-only tool `check_premise({text, surface})` that takes an artifact's question and recommended-focus text and returns (a) the identity claims that bear on it — the KB answer with its citations for a fixed framing query, checked against **a versioned anchor-source list** (the list is the contract, the answer the explanation), (b) the prior decisions that bear on it — one read of the Memory Core graph on the text's problem nouns, through the KB server's existing graph-file path, (c) the concept nodes the text touches, (d) a drift reading: counts of framework-category terms against AI-first terms from two maintained term lists in the repository, with a verdict `aligned` (AI-first terms present, the framing sentence found, at least one anchor source cited), `drifted` (category terms present, no framing sentence), `unknown` (neither). Deterministic and explainable; the three lists are data, versioned beside the tool, and the first instrument, not the judge — the citations are what the author reads. Placement: a tool module beside `describeCollectionStats.mjs` and `ingestSourceFilesTool.mjs`.
2. **Leaf 2 — the receipt in the artifact contracts (neo-agent-skills + the business repository's sitting template):** positioning artifacts — area files' §1 and §5, the record's §2, the pack's documents, README and Introduction edits, blog posts — carry `Premise check: <verdict> · <the KB citations read> · <the MC decision read>`; the `blog-post`, `guide-authoring` and `neo-identity-update` skills name the step in their `references/` payloads (Progressive Disclosure; `SKILL.md` descriptions stay under the 220-byte budget of neo-agent-skills#206); the skills corpus lint and the business repository's commit guard (`ops/hooks/pre-commit`, a second rule in the existing hook) fail an artifact without the line; a day plan's reading rule never suppresses it (the sitting template carries the line). **The adherence instrument:** the count of positioning artifacts with a receipt against those without, per week, reported in the handoff's `## Focus (v1)` section (#957 / #1253) — the number the operator's hint asks for, kept honest by the same probe shape as #14446.
3. **Leaf 3 — the identity concepts' edges (graph):** `json-first-vdom`, `agent-os`, `multi-window`, `multi-threading`, `data-layer` get their `IMPLEMENTED_BY` / `EXPLAINED_BY` / `EXEMPLIFIED_BY` edges from the sources the anchor list names, so the handoff stops listing them as orphans and Golden Path v2 (#122) can rank identity-anchored work.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `check_premise` (KB MCP, new, read-only) | `§neo_identity_anchor`; the KB's `ask_knowledge_base` contract; the KB server's `memoryCoreDbPath` graph read | returns `{claims, citations, anchorsCited, decisions, concepts, counts, verdict}`; never writes | KB unavailable or the graph file unreadable → that part `unknown` with the reason, never a throw | openapi + tool summary | unit arms: the Engine Pro file's §5 reads `drifted` (68/16); the AI-trends file's §5 reads `aligned`; an empty text reads `unknown` |
| the two term lists + the anchor-source list | this ticket; maintained beside the tool | versioned data files; additions by PR | — | the lists' header | a snapshot arm; the anchor list vs the KB's ranking of 2026-10-30 |
| positioning-artifact contracts (skills, the sitting template, the business commit guard) | the skills' authoring guides; the business repository's artifact contract; business PR #23's hook | `Premise check:` line required; the weekly adherence count reported | missing line → the lint fails the artifact | the skills' payloads | the corpus lint arm; the count's probe arm (leaf 2) |
| the five identity concepts | the concept ontology; the anchor list's sources | edges added; the handoff's orphan list shrinks by five | — | the ontology's metadata | the synthesizer's gap audit before/after (leaf 3) |
| `get_memory_core_tool_metrics` `sinceMs` | its own OpenAPI contract | out of scope here; captured as a defect-note (the window does not filter the per-tool counts) | — | — | two calls with different windows, identical totals (2026-10-24 21:5xZ) |

Decision Record impact: aligned-with ADR 0033 (reporting, additive); none amended. The term lists, the anchor list and the verdict are an instrument, never a ranking gate.

## Acceptance Criteria

- [ ] AC-1 `check_premise` on the Engine Pro area file's §1 + §5 text returns `drifted` with the 2026-10-24 counts and the anchor-source citations; on the AI-trends file it returns `aligned` (unit arms on fixtures copied from the texts' public-safe sentences).
- [ ] AC-2 The fixed framing query returns the anchor-source list's entries or their successors, and the tool reports which anchors the KB cited; a KB outage or an unreadable graph file returns `unknown` for that part with a reason.
- [ ] AC-3 (leaf 2) A positioning artifact without the `Premise check:` line fails the skills corpus lint and the business repository's commit guard; the sitting template carries the line.
- [ ] AC-4 (leaf 3) The five concepts carry edges; the next handoff lists none of them as orphaned.
- [ ] AC-5 (post-merge) The next positioning artifact a seat writes carries the receipt and states the AI-first thesis in its recommended focus; receipt on this ticket.
- [ ] AC-6 (leaf 2, post-merge) The weekly adherence count (artifacts with a receipt / all positioning artifacts) appears in the handoff's `## Focus (v1)` section for two consecutive weeks; the first number is the baseline, whatever it is.

## Out of Scope

Changing any price, package or business decision (the lead's and the operator's); rewriting the business documents (flagged to the lead separately); an LLM-judged drift score (a later refinement once the deterministic instrument has a baseline); the Golden Path's ranking (#122); the metrics tool's window defect (a defect-note).

## Avoided Traps

- Banning comparisons: the Engine Pro question asked for them; the check demands the framing sentence, it never forbids the comps.
- A word list as the judge: it is the trigger that makes the author read the citations; the verdict names what is missing, not what is wrong.
- The KB's ranking as the contract: the same question returned a different six on Oct 30 than the five on Oct 24, plus an extrapolation no source states; the anchor list is checked, the answer is read.
- Another prose rule: "MC-first" and "KB first" already exist as prose and did not fire in ten hours of lead-role; this ticket adds a check the artifact's contract demands and a receipt a reviewer sees.
- Blame: the derivation followed the day's rules and the operator's own analogy; the mechanism is what gets fixed.

## Related

#123 (the business engine; the identity concepts are the graph's business half), #1253 (the boot focus read — the sibling anchor at boot) and #957 (`## Focus (v1)`, the surface for AC-6's number), #122 (Golden Path v2), neo #9603 (the object-permanence summary), neo #13144 (the PR review's premise anchors — the review-time sibling), business PR #23 (the commit guard leaf 2 extends), D#10119 and D#10137 (the identity anchors), neo-agent-skills#61 (triggers measurably not firing — the same class), neo-agent-skills#206 (the 220-byte description budget leaf 2 writes under), the `identity-firewall`, `neo-identity-update`, `blog-post` and `guide-authoring` skills (neo-agent-skills), the 2026-10-24 19:40Z handoff orphan list.

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 21:4xZ; no equivalent. A2A in-flight claim sweep: the latest 12 `[lane-claim]` rows, none on this scope. MC sweep: the problem's nouns (the derail into the component-library playbook, the comparison set, object permanence and JSON-first ignored), 6 results — the Oct 23 derivation found and cited above; no prior mechanism decision. Own-assignment sweep: 12 open across the two repositories, none overlapping. Structure map: `ai/mcp/server/knowledge-base` (the tool's home, exists), `ai/services/graph` (the edges), the skills repository (leaf 2); no new directory.

## Intake record (2026-10-30 22:1xZ — self-authored in an earlier session, so the drift probe ran; it fired, so the full gate ran)

- Drift probe (`git log origin/dev --since=2026-10-24T21:00Z`, the ticket's edit surfaces): Brain — `knowledge-base/server.json` + `configBase.mjs` (#1265 manifests, #1356 concept routing), `GoldenPathSynthesizer.mjs` + `focusV1Section.mjs` (#957 / #1333 Focus v1), `probeBusinessMetrics.mjs` (#1368 symlink CLI guard); skills — `blog-post`, `guide-authoring`, `identity-firewall`, `neo-identity-update` (#168 outbound widening, #206 description budget, #225 placement gate). None adds a premise check; the readings are folded above (line refs, the Focus v1 surface, the budget, the KB's graph-file read).
- Prescription checked: `ai/mcp/server/knowledge-base/` (`toolService.mjs` registers the tools; `configBase.mjs` holds `memoryCoreDbPath`) — owns the concern: the KB owns the citations and already reads the decisions' graph; a Brain script in the probe shape would be the lint's fallback, not the author's tool.
- Classification: `valid-as-written`, sharpened in this edit (no premise change). Age: created and last updated 2026-10-24; this repository runs no close-inactive automation, so no bot band applies. ADR successor check: ADR 0018 (engine) and ADR 0033 (Brain) unchanged since 2026-10-24.
- Substrate sufficiency: §The Problem is the enforcement-layer audit — Layer 1 (`§neo_identity_anchor`) loaded and not applied, existing skill logic absent at framing time, no Layer 4 check for positioning artifacts; insufficiency measured by the metrics and the lead's report. Meta-skill: leaf 2 writes into `references/` payloads under the 220-byte description budget. Structural: a new tool module in `ai/mcp/server/knowledge-base/` matches its siblings (`describeCollectionStats.mjs`, `ingestSourceFilesTool.mjs`); leaves 2 and 3 add no `.mjs` file in a new place.
- Sweeps: KB (type `ticket`) — no existing premise-check mechanism for positioning artifacts; the review-time anchors (neo #13144) and the L1–L3 firewall tickets (neo #11241) are the kin. Memory Core (team, 8) — identity-drift corrections on PRs and the ADR 0018 drift rows; no mechanism decision. Tech-debt radar sweeps A and B are these two reads; C/D wait for the leaves.
- Next: a peer read of the Fix and the ledger before the three leaves split (Ada asked, 2026-10-30; Clio as alternate); no leaf is claimed until then.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "category drift premise check identity anchor AI-first engine component library KB citations MC-first non-use orphaned identity concepts adherence count"


## Timeline

- 2026-10-24T21:49:03Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-24T21:49:05Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T21:49:05Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T21:49:05Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-24T21:49:06Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-10-24T21:49:06Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-24T22:41:42Z @neo-opus-ada cross-referenced by #1260

