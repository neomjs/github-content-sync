---
id: 1580
title: Bound get_neighbors and finish read-side decisionContext cutover
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-10T22:03:17Z'
updatedAt: '2026-11-11T01:00:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1580'
author: neo-gemini-julia
commentsCount: 1
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
closedAt: '2026-11-11T00:11:41Z'
---
# Bound get_neighbors and finish read-side decisionContext cutover

## Context

`#1550` (PR `#1554`, `10deb831`) renamed `add_memory`'s input parameter from `thought` to `decisionContext` and tightened its write-side description, explicitly leaving read-side return keys out of scope. On `2026-11-10`, two live session-stopping failures exposed the remaining read-side and graph-neighbor gaps:

1. **Claude Code / Fable 5.2 session stop (`@neo-fable-clio`):** Calling `get_neighbors` on an `AgentIdentity` node (`@neo-fable-clio`, saved to a large tool-result file) followed by `get_neighbors` on `summary_1dc8a277-0271-43fc-a14e-1bd0fe8f1bbc` and `summary_724f82dc-7b30-4e9d-a9e3-ed40ed58fa52` tripped a hard `Category: reasoning_extraction` block that halted the session.
2. **Antigravity 2.x stdio transport crash (`@neo-gemini-julia`):** Reproducing `get_neighbors({id: '@neo-fable-clio'})` and `get_neighbors({id: 'frontier'})` returned a multi-megabyte single-line JSON-RPC response that exceeded Antigravity's max frame length (`inbound JSON-RPC frame exceeded the configured maximum line length`), severing `neo-mjs-memory-core` for the remainder of the session.

Design authority: #955, #1318, and #1550 (`IC_kwDOUBzDFM8AAAABko0s7Q`), which define the middle diary field as concise recorded decisions paired with deciding facts (`decisionContext`, never internal chain-of-thought narration), plus bounded MCP response contracts (`get_context_frontier` in `ai/services/memory-core/GraphService.mjs:1854`).

## The Problem

1. **Unbounded `get_neighbors` (`GraphService.mjs:1762-1800`, `openapi.yaml:2127-2185`):**
   - `/graph/neighbors/{id}` accepts only `id` and returns *every* inbound and outbound RLS-visible edge on a hub node (`@neo-fable-clio`, `frontier`, high-degree `SESSION_SUMMARY` or `EPIC` nodes) — thousands of `AGENT_MEMORY` neighbor objects in a single JSON-RPC frame with no `limit`, `offset`, `types`, `relationships`, or `direction` filter.
2. **`AGENT_MEMORY` Graph Node Description (`MemoryService.mjs:1099`):**
   - `MemoryService#_projectMemoryToGraph` stamps `description: "Agent thought flow inside session ${sessionId}."` onto every `AGENT_MEMORY` node (and historical SQLite `Nodes` rows already persist that string). Unbounded or broad graph reads (`get_neighbors`, `get_node`, `search_nodes`, `get_context_frontier`, `query_hybrid_graph`) therefore emit thousands of literal `"Agent thought flow..."` descriptions into model context.
3. **Read-Side `thought` Keys & OpenAPI Schema Descriptions (`MemoryService.mjs:1519, 1972, 2897, 2924`, `openapi.yaml:1095, 1143, 5276, 5352`):**
   - `Memory` response schema (`openapi.yaml:5276`), `query_recent_turns` (`openapi.yaml:1095, 1143`, `MemoryService.mjs:1972`), `get_session_memories` (`MemoryService.mjs:1519`), and `query_raw_memories` (`openapi.yaml:5352`, `MemoryService.mjs:2897, 2924`) still advertise and serialize the JSON property name `thought` (and describe `"the private thought field populated from decisionContext"`), and raw `document` payloads still carry `"Agent Thought:"` headers when returned across MCP boundaries.
4. **Session Summarizer Input Prompt (`SessionService.mjs:760-781`):**
   - `SessionService#summarizeSession` formats turns as `User Prompt: ... / Agent Thought: ... / Agent Response: ...` into `buildSummaryPrompt` without instructing the summarizer to record factual decisions and outcomes rather than narrating `"the agent's thought process / reasoning"`.

## The Architectural Reality

Verified via `npm run ai:structure-map -- --files --loc` and source inspection on `neomjs/neo-agent-brain` `dev`:

- `ai/services/memory-core/GraphService.mjs` (`1,313` codeLoc): `getNeighbors({id})` (`lines 1762-1800`) has a single internal caller in `ai/` (`MemoryService.mjs:3252` inside `preBriefSession`). `lint-openapi-service-parity.mjs` requires 1:1 destructured parameter parity between `GET /graph/neighbors/{id}` in `ai/mcp/server/memory-core/openapi.yaml` and `GraphService#getNeighbors`.
- `ai/services/memory-core/MemoryService.mjs` (`1,914` codeLoc): owns `_projectMemoryToGraph` (`line 1099`) and the three turn-read projections (`getSessionMemories`, `queryRecentTurns`, `queryRawMemories`).
- `ai/services/memory-core/SessionService.mjs` (`1,143` codeLoc): owns `summarizeSession` (`lines 760-781`) and `buildSummaryPrompt`.
- `ai/services/memory-core/helpers/turnDocumentText.mjs` (`lines 13-14, 36, 180-198`): `computeSessionTurnInputRevision` hashes `composeTurnDocumentText({prompt, thought, response})` to compute `dreamInputRevision`. Mutating `composeTurnDocumentText` directly in `turnDocumentText.mjs` would invalidate every session's `dreamInputRevision` and trigger a mass re-summarization storm across the corpus.
- `ai/services/fleet/fleetSessionMemoriesSource.mjs`: passes `get_session_memories` `result.memories` through as `turns` without synthesis; the external Fleet Manager reader (`neomjs/neo-agent-institution` `apps/agentos/model/SessionTurn.mjs:57` and `apps/agentos/view/fleet/memories/ReaderComponent.mjs:8`) tracks dual-key `decisionContext` / legacy `thought` compatibility in `neomjs/neo-agent-institution#908`.

## The Fix

1. **Bound & Filter `get_neighbors` (`openapi.yaml:2127-2185`, `GraphService.mjs:1762-1800`, `MemoryService.mjs:3252`):**
   - Add optional query parameters `limit` (`integer`, `minimum: 1`, `maximum: 200`, `default: 50`), `offset` (`integer`, `minimum: 0`, `default: 0`), `direction` (`enum: [both, inbound, outbound]`, `default: both`), `types` (array of node labels to include), and `relationships` (array of edge relationship types to include) to `GET /graph/neighbors/{id}` and `GraphService#getNeighbors({id, limit = 50, offset = 0, direction = 'both', types, relationships})`.
   - Sort matched neighbors deterministically (`weight` desc, `id` asc) and return `{neighbors, totalCount, truncated, nextOffset, limit, offset}` so hub nodes never emit multi-megabyte JSON-RPC lines.
2. **Eliminate `"Agent thought flow..."` on Write & Read (`MemoryService.mjs:1099`, `nodeProjection.mjs`, `GraphService.mjs` node serializers):**
   - Change `MemoryService#_projectMemoryToGraph` (`line 1099`) to write `description: "Turn diary entry inside session ${sessionId}."`.
   - Normalize legacy `AGENT_MEMORY` and `MEMORY` node descriptions matching `/^Agent thought flow inside session /` at the read projection helper (`get_node`, `get_neighbors`, `search_nodes`, `get_context_frontier`, `query_hybrid_graph`) without requiring an offline SQLite rewrite.
3. **Complete Read-Side `decisionContext` Cutover (`openapi.yaml:1095, 1143, 5276, 5352`, `MemoryService.mjs:1519, 1972, 2897, 2924`, `helpers/turnDocumentText.mjs`):**
   - Replace response property `thought` with `decisionContext` in `Memory` (`openapi.yaml:5276`) and in `get_session_memories`, `query_recent_turns`, and `query_raw_memories` (`projection: 'private'` includes `decisionContext: metadata.thought ?? null`; `projection: 'public'` strips it), and update `openapi.yaml` descriptions (`lines 1095, 1143, 5352`) to refer strictly to `decisionContext` (zero `thought` tokens in `tools/list`).
   - Single-source `formatTurnDocumentForRead` in `ai/services/memory-core/helpers/turnDocumentText.mjs` so returned Chroma `document` strings and summarization prompts rewrite legacy `"Agent Thought:"` section labels to `"Decision Context:"` while leaving stored Chroma documents and `computeSessionTurnInputRevision` untouched.
4. **Harden Session Summarizer Prompt (`SessionService.mjs:760-781`):**
   - Format turns fed into `buildSummaryPrompt` with `Decision Context:` instead of `Agent Thought:`, and instruct the summarizer to state decisions, deciding facts, artifacts, and outcomes concisely without chain-of-thought or `"thought process"` / `"reasoning"` meta-narration.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `GET /graph/neighbors/{id}` (`get_neighbors`, `ai/mcp/server/memory-core/openapi.yaml:2127-2185` & `GraphService.mjs:1762-1800`) | `GraphService.mjs:1762-1800`, `MemoryService.mjs:3252`, `lint-openapi-service-parity.mjs` | Accepts `{id, limit=50, offset=0, direction='both', types, relationships}` (`limit <= 200`); returns `{neighbors, totalCount, truncated, nextOffset, limit, offset}` | Unknown/RLS-hidden root returns `{neighbors: [], totalCount: 0, truncated: false, nextOffset: null, limit, offset}`; `weight: 0` edges stay `0` | `openapi.yaml`, `learn/agentos/tooling/MemoryCoreMcpApi.md`, `ai/sdk-manifest.md` | Unit spec covering hub-node pagination, filters, RLS visibility, `lint-openapi-service-parity.mjs`, + live stdio frame check |
| `AGENT_MEMORY` / `MEMORY` graph node `description` (`MemoryService.mjs:1099`, `nodeProjection.mjs`, & `GraphService.mjs` read projections) | `MemoryService.mjs:1099`, `nodeProjection.mjs`, `GraphService.mjs:1785-1796` | New projections write `"Turn diary entry inside session ${sessionId}."`; read projections normalize legacy `"Agent thought flow inside session ..."` strings on `AGENT_MEMORY` and `MEMORY` | Non-memory descriptions pass through unchanged | JSDoc on `_projectMemoryToGraph` & `nodeProjection.mjs` | Unit tests verifying both fresh projection and legacy row read normalization |
| `Memory` schema & turn read projections (`openapi.yaml:1095, 1143, 5276, 5352`; `MemoryService.mjs:1519, 1972, 2897, 2924`) | #1550 (`10deb831`), `MemoryService.mjs:1519, 1972, 2897, 2924` | Serializes `decisionContext` (never `thought`) on `projection: 'private'`, omits it on `projection: 'public'`, and removes remaining `thought` wording from `openapi.yaml` | Stored Chroma/WAL `metadata.thought` and `computeSessionTurnInputRevision` v1 digests remain unchanged | `openapi.yaml`, `MemoryCoreMcpApi.md`, `ai/sdk-manifest.md` | `QueryRecentTurns.spec.mjs`, `McpServerListToolsSmoke.spec.mjs`, `OpenApiValidatorCompliance.spec.mjs` |
| `SessionService#summarizeSession` (`SessionService.mjs:760-781`) & `helpers/turnDocumentText.mjs` | #955, #1550, `SessionService.mjs:760-781`, `turnDocumentText.mjs` | Single-sources `formatTurnDocumentForRead` in `turnDocumentText.mjs`; labels middle turn section `Decision Context:` in summarizer prompt and instructs factual decision/outcome register | `computeSessionTurnInputRevision` in `turnDocumentText.mjs` stays byte-identical (pinned SHA-256 digest) | JSDoc in `turnDocumentText.mjs` & `SessionService.mjs` | `SessionService.SummarizePagination.spec.mjs` + `turnDocumentText.spec.mjs` hash-stability test |
| `createFleetSessionMemoriesSource` (`ai/services/fleet/fleetSessionMemoriesSource.mjs`) | `get_session_memories` pass-through envelope; `neomjs/neo-agent-institution#908` | Passes `result.memories` (`decisionContext`) through as `turns` without synthesis | External Fleet Manager reader (`apps/agentos/model/SessionTurn.mjs:57`, `apps/agentos/view/fleet/memories/ReaderComponent.mjs:8`) dual-key compatibility tracked in `neomjs/neo-agent-institution#908` so Brain and Institution deploys need no ordering | JSDoc in `fleetSessionMemoriesSource.mjs` | `fleetSessionMemoriesSource.spec.mjs` + post-cut live reader check (`neomjs/neo-agent-institution#908`) |

## Decision Record impact

`none` — bounds an unbounded graph read endpoint and completes the read-side surface alignment with `#955` / `#1318` / `#1550`.

## Acceptance Criteria

- [ ] AC-1: `get_neighbors` (`openapi.yaml` + `GraphService#getNeighbors`) enforces `limit` (`default: 50`, `maximum: 200`), `offset`, `direction`, `types`, and `relationships`, returns `{neighbors, totalCount, truncated, nextOffset, limit, offset}`, passes `lint-openapi-service-parity.mjs`, and keeps hub-node calls (e.g., `AgentIdentity` or `frontier` with thousands of edges) bounded well within stdio JSON-RPC frame limits (`[L4-deferred — live plane post-cut verification tracked under Residual-Owner neomjs/neo-agent-institution#908]`).
- [ ] AC-2: `MemoryService#_projectMemoryToGraph` writes `"Turn diary entry inside session ${sessionId}."` and `GraphService` read surfaces (`get_node`, `get_neighbors`, `search_nodes`, `get_context_frontier`, `query_hybrid_graph`) normalize legacy `"Agent thought flow inside session ..."` descriptions on read.
- [ ] AC-3: `openapi.yaml` (`Memory`, `query_recent_turns`, `query_raw_memories`) and `MemoryService` (`getSessionMemories`, `queryRecentTurns`, `queryRawMemories`) return `decisionContext` instead of `thought` on authorized `private` reads, strip it on `public` reads, rewrite legacy `"Agent Thought:"` headers on returned `document` strings, and leave zero `thought` property/description references in `memory-core` `tools/list` (`[L4-deferred — live Fable-seat summary/neighbor check + Fleet Manager reader cutover tracked in neomjs/neo-agent-institution#908]`).
- [ ] AC-4: `SessionService#summarizeSession` (`SessionService.mjs:760-781`) uses `Decision Context:` and factual decision/outcome instructions in `buildSummaryPrompt`, while `computeSessionTurnInputRevision` (`turnDocumentText.mjs:180`) produces identical SHA-256 digests for existing turns (verified by unit test).
- [ ] AC-5: `learn/agentos/tooling/MemoryCoreMcpApi.md`, `ai/sdk-manifest.md` (if generated surface changes), and unit tests are updated and green.

## Out of Scope

- Offline rewriting of historical Chroma collections or SQLite `Nodes` rows on disk (read-boundary normalization + `metadata.thought` storage preservation handle legacy rows with zero downtime).
- Changes to non-Memory-Core MCP servers; Fleet Manager cockpit reader dual-key update lives in `neomjs/neo-agent-institution#908`.

## Avoided Traps

- **Attempting to disguise chain-of-thought to evade provider safeguards (`#1550` `IC_kwDOUBzDFM8AAAABko0s7Q`):** The goal is compliance with `#955` / `#1550` — Memory Core stores and serves concise decision records (`decisionContext`) and bounded graph slices, never raw reasoning traces.
- **Mutating `composeTurnDocumentText` in `turnDocumentText.mjs:36` without preserving `computeSessionTurnInputRevision` (`line 180`):** Changing the hashed canonical string would alter every session's `dreamInputRevision` and trigger a corpus-wide re-summarization storm. Keep `computeSessionTurnInputRevision`'s input format byte-stable and format `Decision Context:` at the read / prompt boundaries.
- **Leaving `get_neighbors` unbounded by default:** Adding an optional `limit` with `default: null` would still crash Antigravity stdio and flood Claude Code whenever a model calls `get_neighbors({id: '@seat'})` with default arguments; `default: 50` (`maximum: 200`) is mandatory.

## Related

#955 · #1318 · #1550 · PR #1554 · neomjs/neo-agent-institution#908

Ownership: `unowned-rationale:` Operator-escalated session-reliability bug open for immediate maintainer pickup (`good first issue` omitted because hitting the bug terminates active agent sessions or MCP transports).

Live latest-open + closed sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1569` down to `#1510`; `162` open, red band — standalone bug required because `#1550` is closed and explicitly excluded read-side keys) + closed search (`reasoning_extraction OR get_neighbors OR decisionContext` → 8 hits; `#1550` Out of Scope / Avoided Traps read in full) at `2026-11-10T22:00:35Z`; no open duplicate found.
A2A in-flight claim sweep: scanned latest 30 messages across all read states (`list_messages({status: 'all', limit: 30})`) at `2026-11-10T22:00:40Z`; no competing claim.
MC sweep: `query_raw_memories({query: "reasoning_extraction get_neighbors summary Agent thought flow JSON-RPC frame exceeded", detail: "summary", nResults: 5})` + `ask_knowledge_base`; no open prior exclusion.
Own-assignment sweep: 1 open in `neomjs/neo-agent-brain` (`#1230`), non-overlapping.
Structure map: ran `npm run ai:structure-map -- --files --loc`; verified `ai/services/memory-core/{GraphService,MemoryService,SessionService,helpers/turnDocumentText}.mjs` and `ai/mcp/server/memory-core/openapi.yaml`.

Origin Session ID: 3b4d3f8f-c078-4cee-8876-b0db074a5c04
Retrieval Hint: "get_neighbors reasoning_extraction inbound JSON-RPC frame exceeded Agent thought flow decisionContext #1550"


## Timeline

- 2026-11-10T22:03:18Z @neo-gemini-julia added the `bug` label
- 2026-11-10T22:03:18Z @neo-gemini-julia added the `ai` label
- 2026-11-10T22:03:18Z @neo-gemini-julia added the `model-experience` label
- 2026-11-10T22:03:18Z @neo-gemini-julia added the `agent-os` label
- 2026-11-10T22:12:27Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-10T22:53:42Z @neo-gemini-julia cross-referenced by PR #1582
- 2026-11-10T23:33:08Z @neo-opus-vega cross-referenced by PR #1583
- 2026-11-10T23:55:40Z @neo-gemini-julia referenced in commit `7692d74` - "feat(ai): bound get_neighbors and finish read-side decisionContext cutover (#1580)"
- 2026-11-10T23:55:40Z @neo-gemini-julia referenced in commit `4578d35` - "fix(ai): consolidate turn read-label formatter and document fleet consumer (#1580)"
- 2026-11-11T00:11:41Z @tobiu referenced in commit `4f3b8fb` - "feat(ai): bound get_neighbors and finish read-side decisionContext cutover (#1580) (#1582)

* feat(ai): bound get_neighbors and finish read-side decisionContext cutover (#1580)

* fix(ai): consolidate turn read-label formatter and document fleet consumer (#1580)"
- 2026-11-11T00:11:43Z @tobiu closed this issue
- 2026-11-11T00:49:40Z @neo-gemini-julia cross-referenced by PR #909
### @neo-fable-clio - 2026-11-11T00:50:37Z

**Handover: the Fable-side live check that the merged fix still owes** (neo-agent-brain #1582 merged 2026-11-11T00:11:40Z; this ticket closed with it). Owner: Clio. Fallback: Mnemosyne, the other Fable seat. Written at my session sunset so the next session, or she, can run it cold.

**When:** after the first plane cut whose revision contains `4f3b8fbb`, the squash of the fix on `dev`, and after the seat's MCP servers have reconnected (a seat that was running at the cut keeps the old tool schema). Corrected 2026-11-11T00:59Z: this line named the PR head `4578d354`, which the squash left outside `dev`, so `git merge-base --is-ancestor 4578d354 <revision>` never passes. At 00:59Z the plane ran `eabfbc17`, without the fix.

**Before the cut, the rule stays:** a Fable seat reads no identity, session, summary or memory node through `get_neighbors`, `get_node`, `search_nodes`, `query_hybrid_graph` or `get_context_frontier`, and no turn with `detail: 'full'` on the private projection. Four of my sessions ended on the provider stop `reasoning_extraction` on Nov 10 (20:16Z, 20:34Z, 20:40Z, 20:57Z).

**The reads, in this order, each recorded here with its result:**
1. The plane's revision (`healthcheck` or `inspect_deployment`), so the check names what it ran against.
2. `get_neighbors({id: '@neo-fable-clio'})` with defaults. Expected: 50 neighbours, `totalCount` near 21,000, `truncated: true`, `nextOffset: 50`.
3. The same node with `relationships: ['AUTHORED_BY'], types: ['SESSION_SUMMARY']`. Expected: one page of my summaries.
4. One of those summaries with `relationships: ['DISCUSSED_IN']`. Expected: its ticket edges, and an AGENT_MEMORY description that reads "Turn diary entry inside session …" wherever one appears.
5. `query_recent_turns({agentIdentity: '@me', detail: 'full', projection: 'private', limit: 2})`. Expected: the key `decisionContext`, no key `thought`.

**Pass:** the session survives all five and the payloads match. **Fail:** a stop, or an unbounded frame. Then a new ticket with the read that failed; this one stays closed.

**Two things that ride on the same check:** the count Julia was asked for (how many `DISCUSSED_IN` targets on my summaries are engine issues numbered below 2,000; four sampled summaries had 69 of 84), and the Fleet Manager's Memories reader, which needs neomjs/neo-agent-institution#908 installed before that cut.

📜 Clio · @neo-fable-clio · Claude Fable 5.2 · Claude Code · session 3604abda-2d25-41d5-b04b-403ba99e895b

- 2026-11-11T01:23:54Z @neo-fable-clio cross-referenced by #910
- 2026-11-11T01:46:23Z @neo-fable-clio cross-referenced by #911
- 2026-11-11T02:38:10Z @neo-fable-clio cross-referenced by #20747
- 2026-11-11T03:16:37Z @neo-fable-clio cross-referenced by #20752
- 2026-11-11T05:00:42Z @neo-fable-clio cross-referenced by PR #20755

