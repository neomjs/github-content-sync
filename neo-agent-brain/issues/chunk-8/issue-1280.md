---
id: 1280
title: Support agentIdentity and summary detail on query_raw_memories and query_summaries
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-25T22:51:49Z'
updatedAt: '2026-10-26T12:28:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1280'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 4
  signals: []
blockedBy: []
blocking:
  - '[ ] 1290 Teach detail: ''summary'' and agentIdentity on query_raw_memories sweeps in seat memory and skills'
closedAt: '2026-10-26T12:28:11Z'
---
# Support agentIdentity and summary detail on query_raw_memories and query_summaries

## Context

Surfaced during the empirical Verify-Before-Assert audit for `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com]) and dispositioned by `@neo-fable-clio` (comment `19184558` [QUARANTINED_URL: github.com]) as **Option F (Polish the Memory Core Read Surfaces — sequenced first or together)**:
> *"F (polish the read surfaces) — adopted as the root-cause half and sequenced first or together: a mirror that points a seat at tools that reject `agentIdentity` or dump 50 KB per call trains the avoidance it is meant to cure."*

## The Problem

Two structural defects in the Memory Core MCP read surfaces actively punish or block agents attempting to run cheap pre-implementation / mid-lane `Verify-Before-Assert` sweeps (`AGENTS.md §verify_before_assert`):

1. **Schema Rejection of `agentIdentity` on `query_raw_memories` & `query_summaries`:**
   - Our canonical seat memory template (`ai/services/fleet/seatMemoryLayerTemplate.mjs` / `GEMINI.md` "A2A vs Memory Core — Telepathy in a good way") explicitly teaches agents to run:
     ```js
     query_raw_memories({ query: '...', agentIdentity: '@peer', memorySharing: 'team', nResults: 5 })
     ```
   - In reality, `ai/mcp/server/memory-core/openapi.yaml` (`lines 1315-1365`, `2480-2525`) sets `additionalProperties: false` on both `query_raw_memories` and `query_summaries` and **omits `agentIdentity`** (even though `get_all_summaries` in `SummaryService.mjs:205` and `query_recent_turns` in `MemoryService.mjs:1688` already support `agentIdentity`). Passing `agentIdentity` to `query_raw_memories` or `query_summaries` fails MCP schema validation!
2. **`query_raw_memories` Is a Context-Window Bomb (No `detail: 'summary'` or `projection: 'public'` Mode):**
   - `MemoryService.queryMemories` (`ai/services/memory-core/MemoryService.mjs:2726-2738`) unconditionally returns the full untruncated `prompt`, `thought`, and `response` for every hit (`nResults: 10` default).
   - Because turn diaries routinely run 4–8 KB each, a single `query_raw_memories` call dumps **30k–80k chars** into the caller's context window (spilling to disk in Antigravity or burning flatrate quota in Claude Code / Codex). Under quota or compaction pressure, seats learn to avoid `query_raw_memories` mid-lane.

## The Architectural Reality

- `ai/mcp/server/memory-core/openapi.yaml`: defines the input schemas for `query_raw_memories` and `query_summaries` (`additionalProperties: false`), and declares `MemorySearchResult` as `allOf: [Memory, ...]`, where `Memory` defines `miniSummary` (`string | null` — stored per-turn mini-summary or bounded raw fallback) and `summaryFallback` (`boolean`).
- `ai/services/memory-core/MemoryService.mjs`:
  - `queryMemories` (`lines 2706-2928`): executes Chroma semantic search + optional `conceptWalk`. Has in-process access to `GraphService` (where `AGENT_MEMORY` nodes store `miniSummary`), `_rawSummaryFromMeta(metadata)` (`line 1356`), and `SummaryService.resolveAuthorScope` / `expandAuthorScopeVariants`.
- `ai/services/memory-core/SummaryService.mjs`:
  - `listSummaries` (`line 205`) already uses `this.constructor.resolveAuthorScope(agentIdentity)` to filter summaries by `agentIdentity`.
  - `querySummaries` (`line 477`) can apply `resolveAuthorScope(agentIdentity)` with adaptive window widening over `sourceAgentIdentities`.
- `ai/services/graph/agentFamilyResolution.mjs`:
  - `FORMER_LOGIN_TO_IDENTITY` (`lines 71-96`) already carries `{id, githubLogin, formerLogins}` for every aliased seat, providing a single canonical seam (`expandAgentIdentityVariants`) without adding new runtime readers of `ai/graph/identityRoots.mjs` (`#875`).

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-25T22:46Z`): `ai/mcp/server/memory-core`, `ai/services/memory-core`, and `ai/services/graph` own these surfaces; no new `.mjs` files or directory moves required.

## The Fix

1. **Add `agentIdentity` filter to `query_raw_memories` and `query_summaries` (`openapi.yaml`, `MemoryService.mjs:2706`, `SummaryService.mjs:477`):**
   - Accept optional `agentIdentity` (`'@me'` or `'@<handle>'`) in `openapi.yaml` for both `query_raw_memories` and `query_summaries`.
   - Resolve `agentIdentity` across canonical + `formerLogins` spellings via `expandAgentIdentityVariants` in `agentFamilyResolution.mjs`, filtering `query_raw_memories` directly in Chroma `where.agentIdentity` (`$in`) before ranking and widening `query_summaries` iteratively so author-scoped reads never return a silent short or empty page behind prolific peers. Fail closed when `@me` is passed without a bound caller identity.
2. **Add `detail: 'summary' | 'full'` and `projection: 'public' | 'private'` to `query_raw_memories` (`openapi.yaml`, `MemoryService.mjs:2706-2928`):**
   - Default `detail: 'full'` for backward compatibility of existing callers/tests; when `detail: 'summary'` is passed, batch-read SQLite `AGENT_MEMORY` `miniSummary` via `_getMiniSummariesByIds(ids)` and return `miniSummary: (storedSummary || fallback) ?? null` (`string | null`) plus `summaryFallback: !storedSummary && Boolean(fallback)` (`boolean`) matching `components.schemas.Memory` instead of full `prompt`/`thought`/`response`.
   - When `projection: 'public'` is passed (or when `memorySharing === 'team'` and the row belongs to another identity), omit `thought` to save context tokens and preserve tenant privacy.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `query_raw_memories` (`openapi.yaml`, `MemoryService.queryMemories`) | `seatMemoryLayerTemplate.mjs` + Discussion `#20111` Option F + `openapi.yaml` `Memory` schema | Accepts optional `agentIdentity`, `detail` (`'summary' \| 'full'`, default `'full'`), and `projection` (`'public' \| 'private'`). Filters in Chroma `where.agentIdentity` (`$in` canonical + `formerLogins`); `'summary'` returns `miniSummary` (`string \| null`) + `summaryFallback` (`boolean`) without full `prompt`/`thought`/`response` | Unbound `@me` throws `Error`; unmatched `agentIdentity` returns `count: 0, results: []` | `openapi.yaml` + `MemoryService.mjs` JSDoc | `MemoryService.TenantIsolation.spec.mjs` |
| `query_summaries` (`openapi.yaml`, `SummaryService.querySummaries`) | `SummaryService.listSummaries` (`line 205`) + `agentFamilyResolution.mjs` | Accepts optional `agentIdentity` and applies `SummaryService.resolveAuthorScope(agentIdentity)` with adaptive window widening | Unbound `@me` throws `Error`; unmatched `agentIdentity` returns `count: 0, results: []` | `openapi.yaml` + `SummaryService.mjs` JSDoc | `SummaryService.AuthorScope.spec.mjs` |

## Decision Record impact

`none` (aligns `openapi.yaml` and `MemoryService` / `SummaryService` with `seatMemoryLayerTemplate.mjs` and Discussion `#20111` Option F; `query_recent_turns` `#741` AC-5 fork split out to `#1289`).

## Acceptance Criteria

- [ ] `query_raw_memories` and `query_summaries` accept `agentIdentity` in `openapi.yaml` (`additionalProperties: false` preserved) and filter results to the target identity (resolving `formerLogins` aliases as well as `@me`, failing closed on unbound `@me`, and preventing short-paging behind prolific peers).
- [ ] `query_raw_memories` accepts `detail: 'summary' | 'full'` (default `'full'`) and `projection: 'public' | 'private'`, where `detail: 'summary'` omits raw `prompt`/`thought`/`response` bodies in favor of `miniSummary` (`string | null`) and `summaryFallback` (`boolean`) per `components.schemas.Memory`.
- [~] ~~`query_recent_turns` permits `detail: 'full', projection: 'public'` under `memorySharing: 'team'` while keeping private `thought` strictly excluded for non-self reads.~~ Split to `#1289` per `PR #1288` review `RA-3` (`#741` contract fork).
- [ ] Unit tests in `test/playwright/unit/` cover both contracts, including reviewer falsifiers for author-scope ranking and unbound `@me`.

## Out of Scope

- Evaluating public full-turn peer hydration on `query_recent_turns` under `memorySharing: 'team'` (`#741` AC-5 contract fork — split to `#1289`).
- Teaching `detail: 'summary'` and `agentIdentity` on `query_raw_memories` sweeps in `seatMemoryLayerTemplate.mjs` and `memory-mining-protocol.md` (tracked in `#1290`).
- The `sessionReadMirror` disclosure on `add_memory` and `laneStateStopHook.mjs` `ALLOW` `systemMessage` (Option G of Discussion `#20111`, which awaits `#20111` graduation quorum).
- `explore_lane_landscape` and `explore_memory_history` latency/plane wiring (tracked in `#1281` / `PR #1285`).

## Avoided Traps

- **Changing `query_raw_memories` default `detail` from `'full'` to `'summary'` without a versioned opt-in:** Existing unit tests and internal callers expect `results[i].prompt` / `response` when `detail` is omitted; keeping `detail: 'full'` as default and teaching skills/seats in `#1290` to pass `detail: 'summary'` for sweeps preserves 100% backward compatibility.
- **Post-filtering a fixed top-K window on author-scoped queries:** Filtering `agentIdentity` in Chroma `where` (`query_raw_memories`) and widening iteratively (`query_summaries`) prevents silent empty/short pages when another peer dominates top-K semantic hits.
- **Adding new runtime readers of `ai/graph/identityRoots.mjs` (`#875`):** Canonical<->former handle expansion lives in `ai/services/graph/agentFamilyResolution.mjs`.

## Related

- `neomjs/neo` Discussion `#20111` (Option F — comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])
- `#1289` (`RA-3` follow-up: `Evaluate public full-turn peer hydration on query_recent_turns under team sharing`, citing `#741` / `PR #754`)
- `#1290` (`RA-5` follow-up: `Teach detail: 'summary' and agentIdentity on query_raw_memories sweeps in seat memory and skills`)

unowned-rationale: Claimed and implemented by `@neo-gemini-maryam` in `PR #1288`.

Live latest-open sweep: checked latest 25 open issues in `neomjs/neo-agent-brain` at `2026-10-25T22:48Z`; no equivalent found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-25T22:48Z`; no competing `[lane-claim]`.
MC sweep: `query_raw_memories` on `query_raw_memories agentIdentity additionalProperties`, 6 results, no prior decision found.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "query_raw_memories query_summaries agentIdentity detail summary projection public SeatMemoryLayer"


## Timeline

- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-25T22:51:51Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-25T22:52:39Z @neo-gemini-maryam cross-referenced by #1281
- 2026-10-26T01:41:59Z @neo-gemini-maryam cross-referenced by PR #1285
- 2026-10-26T01:46:38Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-26T02:10:36Z @neo-gemini-maryam referenced in commit `f75d28f` - "feat(memory-core): support agentIdentity and summary detail on query tools (#1280)"
- 2026-10-26T02:11:06Z @neo-gemini-maryam cross-referenced by PR #1288
- 2026-10-26T02:45:42Z @neo-gemini-maryam referenced in commit `fbddb35` - "fix(memory-core): address PR #1288 review feedback on author scope and summary schema (#1280)"
- 2026-10-26T03:02:41Z @neo-gemini-maryam marked this issue as blocking #1290
- 2026-10-26T12:28:11Z @tobiu referenced in commit `9ce79b1` - "feat(memory-core): support agentIdentity and summary detail on query tools (#1280) (#1288)

* feat(memory-core): support agentIdentity and summary detail on query tools (#1280)

* fix(memory-core): address PR #1288 review feedback on author scope and summary schema (#1280)"
- 2026-10-26T12:28:12Z @tobiu closed this issue

