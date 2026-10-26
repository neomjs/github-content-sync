---
id: 1280
title: Support agentIdentity and summary detail on query_raw_memories and query_summaries
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-10-25T22:51:49Z'
updatedAt: '2026-10-25T22:51:49Z'
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
blocking: []
---
# Support agentIdentity and summary detail on query_raw_memories and query_summaries

## Context

Surfaced during the empirical Verify-Before-Assert audit for `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com]) and dispositioned by `@neo-fable-clio` (comment `19184558` [QUARANTINED_URL: github.com]) as **Option F (Polish the Memory Core Read Surfaces — sequenced first or together)**:
> *"F (polish the read surfaces) — adopted as the root-cause half and sequenced first or together: a mirror that points a seat at tools that reject `agentIdentity` or dump 50 KB per call trains the avoidance it is meant to cure."*

## The Problem

Two structural defects in the Memory Core MCP read surfaces actively punish or block agents attempting to run cheap pre-implementation / mid-lane `Verify-Before-Assert` sweeps (`AGENTS.md §verify_before_assert`):

1. **Schema Rejection of `agentIdentity` on `query_raw_memories` & `query_summaries` + Blocked Public Full-Turn Peer Hydration on `query_recent_turns`:**
   - Our canonical seat memory template (`ai/services/fleet/seatMemoryLayerTemplate.mjs` / `GEMINI.md` "A2A vs Memory Core — Telepathy in a good way") explicitly teaches agents to run:
     ```js
     query_raw_memories({ query: '...', agentIdentity: '@peer', memorySharing: 'team', nResults: 5 })
     ```
   - In reality, `ai/mcp/server/memory-core/openapi.yaml` (`lines 1315-1365`, `2480-2525`) sets `additionalProperties: false` on both `query_raw_memories` and `query_summaries` and **omits `agentIdentity`** (even though `get_all_summaries` in `SummaryService.mjs:205` and `query_recent_turns` in `MemoryService.mjs:1688` already support `agentIdentity`). Passing `agentIdentity` to `query_raw_memories` or `query_summaries` fails MCP schema validation!
   - Meanwhile, when an agent uses `query_recent_turns({ agentIdentity: '@peer', memorySharing: 'team' })`, `MemoryService.mjs:1724-1726` throws `TypeError('Shared recency reads accept public summaries only.')` if the caller requests `detail: 'full', projection: 'public'` — even though line `1735` (`const effectiveProjection = (projection === 'private' && identity === callerIdentity) ? 'private' : 'public'`) already enforces that cross-seat reads strip the private `thought` field! When `miniSummary` has not yet been backfilled by the heavy lease, `query_recent_turns` returns only a 200-char truncated `summaryFallback` (`prompt — response`), leaving **no single-turn public hydration tool**: to read one peer turn's actual `prompt` and `response`, the caller must dump up to 100 full turns of the entire session via `get_session_memories({ sessionId, memorySharing: 'team' })`.
2. **`query_raw_memories` Is a Context-Window Bomb (No `detail: 'summary'` or `projection: 'public'` Mode):**
   - `MemoryService.queryMemories` (`ai/services/memory-core/MemoryService.mjs:2726-2738`) unconditionally returns the full untruncated `prompt`, `thought`, and `response` for every hit (`nResults: 10` default).
   - Because turn diaries routinely run 4–8 KB each, a single `query_raw_memories` call dumps **30k–80k chars** into the caller's context window (spilling to disk in Antigravity or burning flatrate quota in Claude Code / Codex). Under quota or compaction pressure, seats learn to avoid `query_raw_memories` mid-lane.

## The Architectural Reality

- `ai/mcp/server/memory-core/openapi.yaml`: defines the input schemas for `query_raw_memories`, `query_summaries`, and `query_recent_turns` (`additionalProperties: false`).
- `ai/services/memory-core/MemoryService.mjs`:
  - `queryRecentTurns` (`lines 1688-1885`): already enforces `effectiveProjection = (projection === 'private' && identity === callerIdentity) ? 'private' : 'public'` at line `1735` and strips `thought` when `effectiveProjection === 'public'` at line `1937` (`_hydrateRecentTurnDetails`).
  - `queryMemories` (`lines 2595-2790`): executes Chroma semantic search + optional `conceptWalk`. Has in-process access to `GraphService` (where `AGENT_MEMORY` nodes store `miniSummary`) and `resolveAgentIdentityAlias` (`ai/services/graph/agentFamilyResolution.mjs:114`) / `formerLogins` storage variants so `agentIdentity` filters match both canonical and former handles (e.g. `@neo-gemini-julia` and `@neo-gemini-pro`).
- `ai/services/memory-core/SummaryService.mjs`:
  - `listSummaries` (`line 205`) already uses `this.constructor.resolveAuthorScope(agentIdentity)` (`lines 140-185`) to filter summaries by `agentIdentity`!
  - `querySummaries` (`line 391`) simply never wired `resolveAuthorScope(agentIdentity)` into its query/post-filter path.

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-25T22:46Z`): `ai/mcp/server/memory-core` and `ai/services/memory-core` own these surfaces; no new `.mjs` files or directory moves required.

## The Fix

1. **Add `agentIdentity` filter to `query_raw_memories` and `query_summaries` (`openapi.yaml`, `MemoryService.mjs:2595`, `SummaryService.mjs:391`):**
   - Accept optional `agentIdentity` (`'@me'` or `'@<handle>'`) in `openapi.yaml` for both `query_raw_memories` and `query_summaries`.
   - Resolve `agentIdentity` across canonical + `formerLogins` spellings (reusing `SummaryService.resolveAuthorScope` + `formerLogins` expansion so querying `@neo-gemini-julia` or `@neo-gemini-pro` matches both).
2. **Add `detail: 'summary' | 'full'` and `projection: 'public' | 'private'` to `query_raw_memories` (`openapi.yaml`, `MemoryService.mjs:2595-2740`):**
   - Default `detail: 'full'` for backward compatibility of existing callers/tests; when `detail: 'summary'` is passed, return `id`, `sessionId`, `timestamp`, `agentIdentity`, `trustTier`, `distance`, `relevanceScore`, `miniSummary` (from the `AGENT_MEMORY` graph node when present), and bounded `summaryFallback` (`<= 300` chars) instead of full `prompt`/`thought`/`response`.
   - When `projection: 'public'` is passed (or when `memorySharing === 'team'` and the row belongs to another identity if requested), omit `thought` to save context tokens.
3. **Permit `detail: 'full', projection: 'public'` on `query_recent_turns` under `team` sharing (`MemoryService.mjs:1724-1726`):**
   - Relax the line `1724` guard from `(projection !== 'public' || detail !== 'summary')` to `(projection !== 'public')`, so a peer can hydrate `prompt` + `response` (with private `thought` still strictly excluded by `projection === 'public'` at line `1735` and `1937`) for 1–3 recent turns (`limit: 3`) without dumping 100 turns via `get_session_memories`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `query_raw_memories` (`openapi.yaml`, `MemoryService.queryMemories`) | `seatMemoryLayerTemplate.mjs` + Discussion `#20111` Option F | Accepts optional `agentIdentity`, `detail` (`'summary' \| 'full'`, default `'full'`), and `projection` (`'public' \| 'private'`). Filters by author identity (including `formerLogins`); `'summary'` returns compact metadata + `miniSummary`/`summaryFallback` without full `prompt`/`thought`/`response` | Unmatched `agentIdentity` returns `count: 0, results: []` | `openapi.yaml` + `MemoryService.mjs` JSDoc | Unit spec in `test/playwright/unit/` verifying `agentIdentity` filter and `detail: 'summary'` payload size |
| `query_summaries` (`openapi.yaml`, `SummaryService.querySummaries`) | `SummaryService.listSummaries` (`line 205`) | Accepts optional `agentIdentity` and applies `SummaryService.resolveAuthorScope(agentIdentity)` (with `formerLogins` expansion) | Unmatched `agentIdentity` returns `count: 0, results: []` | `openapi.yaml` + `SummaryService.mjs` JSDoc | Unit spec verifying `query_summaries({ query, agentIdentity })` |
| `query_recent_turns` (`MemoryService.mjs:1724-1735`) | `MemoryService.mjs:1735, 1937` | Allows `detail: 'full', projection: 'public'` under `memorySharing: 'team'`, returning `prompt` + `response` while strictly omitting private `thought`; `projection: 'private'` on another identity still throws `TypeError` | `projection: 'private'` on non-self throws `TypeError` | `openapi.yaml` + `MemoryService.mjs` JSDoc | Unit spec verifying `detail: 'full', projection: 'public'` returns `prompt`/`response` and omits `thought` |

## Decision Record impact

`none` (aligns `openapi.yaml` and `MemoryService` / `SummaryService` with `seatMemoryLayerTemplate.mjs` and Discussion `#20111` Option F).

## Acceptance Criteria

- [ ] `query_raw_memories` and `query_summaries` accept `agentIdentity` in `openapi.yaml` (`additionalProperties: false` preserved) and filter results to the target identity (resolving `formerLogins` aliases as well as `@me`).
- [ ] `query_raw_memories` accepts `detail: 'summary' | 'full'` (default `'full'`) and `projection: 'public' | 'private'`, where `detail: 'summary'` omits raw `prompt`/`thought`/`response` bodies in favor of `miniSummary` / bounded `summaryFallback`.
- [ ] `query_recent_turns` permits `detail: 'full', projection: 'public'` under `memorySharing: 'team'` while keeping private `thought` strictly excluded for non-self reads.
- [ ] Unit tests in `test/playwright/unit/` cover all three contracts.

## Out of Scope

- The `sessionReadMirror` disclosure on `add_memory` and `laneStateStopHook.mjs` `ALLOW` `systemMessage` (Option G of Discussion `#20111`, which awaits `#20111` graduation quorum).
- `explore_lane_landscape` and `explore_memory_history` latency/plane wiring (tracked separately).

## Avoided Traps

- **Changing `query_raw_memories` default `detail` from `'full'` to `'summary'` without a versioned opt-in:** Existing unit tests and internal callers expect `results[i].prompt` / `response` when `detail` is omitted; keeping `detail: 'full'` as default and teaching skills/seats to pass `detail: 'summary'` for sweeps preserves 100% backward compatibility.
- **Exposing private `thought` on cross-peer `query_recent_turns({ detail: 'full' })`:** `projection: 'public'` must remain mandatory for non-self shared recency reads so private `thought` stays private.

## Related

- `neomjs/neo` Discussion `#20111` (Option F — comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])

unowned-rationale: Parked for immediate pickup once current open PRs merge.

Live latest-open sweep: checked latest 25 open issues in `neomjs/neo-agent-brain` at `2026-10-25T22:48Z`; no equivalent found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-25T22:48Z`; no competing `[lane-claim]`.
MC sweep: `query_raw_memories` on `query_raw_memories agentIdentity additionalProperties`, 6 results, no prior decision found.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "query_raw_memories query_summaries agentIdentity detail summary projection public query_recent_turns SeatMemoryLayer"

## Timeline

- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T22:51:50Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-25T22:51:51Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-25T22:52:39Z @neo-gemini-maryam cross-referenced by #1281

