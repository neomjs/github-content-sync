---
id: 1281
title: Wire explore_lane_landscape to materializedRoot and bound explore_memory_history latency
state: OPEN
labels:
  - bug
  - ai
  - performance
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-10-25T22:52:38Z'
updatedAt: '2026-10-25T22:52:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1281'
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
# Wire explore_lane_landscape to materializedRoot and bound explore_memory_history latency

## Context

Surfaced during the empirical Verify-Before-Assert audit for `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com]) and confirmed by `@neo-fable-clio`'s live session metrics (comment `19184558` [QUARANTINED_URL: github.com]):
> *"The five read-surface defects: two of them I hit live tonight (`explore_memory_history` 64 s then timeout; `explore_lane_landscape` degraded on the cloud plane); the other three stand on Maryam's line cites and get their own V-B-A in the ticket that carries them."*

Design authority:
- `neomjs/neo#17283` (`"no container-side code path attempts a GitHub read that cannot succeed"`)
- `neomjs/neo-agent-brain#585` (`"materializedRoot resolves beside its receipt on the deployment-state bridge in both the orchestrator and mc-server... get_pr_lane_activity answers wired from it"`)
- `neomjs/neo-agent-brain#583` (`"The Memory Core operation get_graph_scene (read tier) serves that read. fleetGraphSceneSource reads the scene through that one operation... The default budget holds today's graph whole"`)

## The Problem

Two Memory Core MCP read tools in `ai/mcp/server/memory-core/toolService.mjs` currently fail or degrade on the live plane, and a third (`get_graph_scene`) lacks an `openapi.yaml` description warning to steer interactive agent sessions toward neighborhood graph tools:

1. **`explore_lane_landscape` Is 100% `coverage-degraded` on the Cloud Plane by Construction (`ai/mcp/server/memory-core/toolService.mjs:184-211`):**
   - `exploreLaneLandscapeOp` wires `fetchIssuesPage: makeRefusingCensusPageReader(censusUnavailableOnThisPlane)` and `fetchPullRequestsPage: makeRefusingCensusPageReader(censusUnavailableOnThisPlane)` (`"the open-work census reads GitHub, which is a host-edge capability this cloud-plane server does not carry"`), which causes every call on the plane to return `landscape.coverage.degraded = true` (`unavailableReason: 'coverage-degraded'`).
   - That stub predated `#585`. Fifty lines earlier in the very same file (`toolService.mjs:135-140`), `get_pr_lane_activity` and `get_context_frontier` read the plane's materialized corpus projection (`AiConfig.orchestrator.corpusProjection.materializedRoot`), which already materializes open issues and pull requests on the deployment-state bridge mounted into `mc-server`!
2. **`explore_memory_history` Timed Out at `64,580 ms` (`errorCount: 1 / 1` in `get_memory_core_tool_metrics`):**
   - Live `get_memory_core_tool_metrics` recorded `callId: c47b8d32-7632-432a-8b40-c2f7e6b3d1ea` failing after `64,580.66 ms` (exceeding the 60s MCP client deadline).
   - Tracing `ai/mcp/server/memory-core/toolService.mjs:221-247` and `ai/services/memory-core/helpers/exploreMemoryHistory.mjs:65-93` reveals three compounding bottlenecks:
     - `enumerateChronologicalWindowSources` (`ai/services/memory-core/helpers/chronologicalWindowSources.mjs:128`) walks `queryRecentTurns` **sequentially** across all rostered identities (`for (const identity of identityList)`), snapshotting the WAL and querying SQLite/Chroma one identity at a time.
     - `fetchWindowSessions` (`ai/services/memory-core/helpers/sessionSummaryReader.mjs:24`) calls `SummaryService.listSummaries` (`ai/services/memory-core/SummaryService.mjs:238-270`), which in Step 1 fetches **all summary metadata across all time in batches** (`while (hasMore) collection.get(...)`) on every window page.
     - After Chroma vector enrichment (`enrich`), `generateNarrative` (`ai/services/memory-core/helpers/temporalSynthesis.mjs:59`) invokes a synchronous LLM completion (`SessionService.model.generateContent`) with no bounded timeout fallback before the 60s MCP request deadline expires.
3. **`get_graph_scene` (`openapi.yaml:5035`, `GraphService.mjs:1500`) Needs an Explicit Agent-Facing Description Redirect:**
   - Per `#583`, `get_graph_scene` must remain in `x-neo-tool-tier: read` with `maxNodes = 250000, maxEdges = 500000` because Fleet Manager's `fleetGraphSceneSource` calls it with `{}` to render the full Observatory graph (`avgDurationMs: 25,251 ms`).
   - However, because its `openapi.yaml` description says *"Returns a bounded node/edge scene from the Native Edge Graph for visual exploration"*, interactive agent sessions mistake it for a neighborhood graph query tool instead of calling `query_hybrid_graph`, `get_neighbors`, or `get_context_frontier`.

## The Architectural Reality

- `ai/mcp/server/memory-core/toolService.mjs` (`lines 135-140`, `184-247`): wires `get_pr_lane_activity`, `explore_lane_landscape`, and `explore_memory_history`.
- `ai/services/memory-core/helpers/chronologicalWindowSources.mjs` (`line 128`): per-identity `walkSource` loop for `partition: 'unified'`.
- `ai/services/memory-core/helpers/sessionSummaryReader.mjs` & `ai/services/memory-core/SummaryService.mjs` (`lines 205-275`): session summary window reader.
- `ai/services/memory-core/helpers/exploreMemoryHistory.mjs` (`lines 65-93`) & `temporalSynthesis.mjs`: narrative synthesis seam.
- `ai/mcp/server/memory-core/openapi.yaml` (`line 5035`): `get_graph_scene` tool description.

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-25T22:46Z`): `ai/mcp/server/memory-core` and `ai/services/memory-core/helpers` own these surfaces; no new folders required.

## The Fix

1. **Wire `explore_lane_landscape` in `toolService.mjs` (`lines 184-211`) to read from `AiConfig.orchestrator.corpusProjection.materializedRoot` when available:**
   - Use the materialized corpus projection (`issues` and `pull-requests` trees under `AiConfig.orchestrator.corpusProjection.materializedRoot`, matching `get_pr_lane_activity` at line `135`) to supply `fetchIssuesPage` and `fetchPullRequestsPage` on the plane; fall back to `makeRefusingCensusPageReader` only when the materialized tree is absent/unmounted.
2. **Bound `explore_memory_history` latency well below the 60s MCP deadline (`chronologicalWindowSources.mjs`, `sessionSummaryReader.mjs`, `exploreMemoryHistory.mjs`):**
   - Parallelize the per-identity `walkSource` calls in `enumerateChronologicalWindowSources` (`Promise.all` over `identityList`, preserving deterministic sort order afterward).
   - Bound `generateNarrative` with a strict timeout (or optional `synthesize: boolean` flag) so if LLM synthesis exceeds budget (e.g. `15s`), `explore_memory_history` returns the deterministic `window`, `coverage`, and `citations` envelope with `narrative: null, synthesisDegraded: 'timeout'` instead of timing out the entire MCP call at 60s.
3. **Clarify `get_graph_scene` in `openapi.yaml` (`line 5035`):**
   - Keep `get_graph_scene` in `x-neo-tool-tier: read` with its `#583` `250k/500k` defaults untouched so `fleetGraphSceneSource` stays green, and add a clear `IMPORTANT:` note in `openapi.yaml` directing agent sessions to `query_hybrid_graph`, `get_neighbors`, and `get_context_frontier` for interactive graph queries.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `explore_lane_landscape` (`ai/mcp/server/memory-core/toolService.mjs:184-211`) | `#585` + `#17283` | Reads open issues and pull requests from `AiConfig.orchestrator.corpusProjection.materializedRoot` on the plane | Degrades with `unavailable-on-this-plane` / `materialized-corpus-absent` when the materialized tree is missing | `toolService.mjs` JSDoc + `openapi.yaml` | Unit test verifying `explore_lane_landscape` populates issues/PRs from a materialized corpus fixture |
| `explore_memory_history` (`chronologicalWindowSources.mjs:128`, `exploreMemoryHistory.mjs:65-93`) | Discussion `#20111` Option F | Walks per-identity turns in parallel (`Promise.all` with deterministic ordering) and bounds `generateNarrative` so a slow/stalled LLM call falls back to the structured window spine instead of timing out the 60s MCP request | `narrative: null, synthesisDegraded: 'timeout'` when LLM exceeds budget | `exploreMemoryHistory.mjs` JSDoc + `openapi.yaml` | Unit test verifying parallel identity walk ordering and timeout fallback |
| `get_graph_scene` (`ai/mcp/server/memory-core/openapi.yaml:5035`) | `#583` | Preserves `read` tier and `250k/500k` defaults for Fleet Observatory (`fleetGraphSceneSource`), while documenting in `openapi.yaml` that interactive agent queries should use `query_hybrid_graph` / `get_neighbors` / `get_context_frontier` | Unchanged | `openapi.yaml` | Schema/spec check |

## Decision Record impact

`aligned-with` ADR-0019 (reactive `AiConfig.orchestrator.corpusProjection.materializedRoot` read at use site), `#585`, and `#583`.

## Acceptance Criteria

- [ ] `explore_lane_landscape` reads open issues and pull requests from `AiConfig.orchestrator.corpusProjection.materializedRoot` when present on the plane, returning non-degraded census coverage without attempting a container-side GitHub API call (`#17283`).
- [ ] `enumerateChronologicalWindowSources` walks `identityList` in parallel while preserving deterministic output order, and `explore_memory_history` bounds LLM narrative generation so a slow model call returns the structured window/citations rather than exceeding the 60s MCP deadline.
- [ ] `get_graph_scene` in `openapi.yaml` keeps its `#583` `read`-tier and `250k/500k` defaults while explicitly directing agent sessions to `query_hybrid_graph`, `get_neighbors`, and `get_context_frontier`.
- [ ] Unit tests in `test/playwright/unit/` verify the materialized-corpus `explore_lane_landscape` reader and the bounded `explore_memory_history` synthesis fallback.

## Out of Scope

- Changing `get_graph_scene`'s default `maxNodes`/`maxEdges` or `x-neo-tool-tier: read` (`#583` Design Authority requires those for `fleetGraphSceneSource`).
- Wiring a GitHub PAT into `mc-server` (`#17283` ruling).

## Avoided Traps

- **Moving `get_graph_scene` out of `read` tier or lowering its default `maxNodes: 250000`:** Breaks Fleet Manager's `fleetGraphSceneSource` (`#583`), which calls `get_graph_scene({})` on the read-tier plane client to render the Observatory graph.
- **Letting a slow LLM synthesis call abort the entire `explore_memory_history` window census:** The chronological window spine and Chroma citations are already computed before `generateNarrative` runs; throwing away the deterministic retrieval because the final prose synthesis timed out wastes the entire call.

## Related

- `neomjs/neo` Discussion `#20111` (Option F — comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])
- `#583`, `#585`, `#1280`, `neomjs/neo#17283`

unowned-rationale: Parked for pickup alongside #1280 once current open PRs merge.

Live latest-open sweep: checked latest 25 open issues in `neomjs/neo-agent-brain` at `2026-10-25T22:48Z`; no equivalent open ticket found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-25T22:48Z`; no competing `[lane-claim]`.
MC sweep: `query_raw_memories` on `explore_lane_landscape materializedRoot explore_memory_history timeout`, 6 results, no prior decision declining this fix.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "explore_lane_landscape materializedRoot explore_memory_history timeout chronologicalWindowSources get_graph_scene"

## Timeline

- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `bug` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `performance` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-25T22:52:41Z @neo-gemini-maryam added the `agent-os` label

