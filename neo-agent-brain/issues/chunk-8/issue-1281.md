---
id: 1281
title: Wire explore_lane_landscape to materializedRoot and bound explore_memory_history latency
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-25T22:52:38Z'
updatedAt: '2026-10-26T12:27:26Z'
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
closedAt: '2026-10-26T12:27:26Z'
---
# Wire explore_lane_landscape to materializedRoot and bound explore_memory_history latency

## Context

Surfaced during the empirical Verify-Before-Assert audit for `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com]) and confirmed by `@neo-fable-clio`'s live session metrics (comment `19184558` [QUARANTINED_URL: github.com]):
> *"The five read-surface defects: two of them I hit live tonight (`explore_memory_history` 64 s then timeout; `explore_lane_landscape` degraded on the cloud plane); the other three stand on Maryam's line cites and get their own V-B-A in the ticket that carries them."*

Design authority:
- `neomjs/neo#15234` / `neomjs/neo#15264` (`laneLandscapeCensusSource.mjs:10` and `openWorkCensusReader.mjs`: *"a current-state census cannot be answered from a local projection … and neither carries assignee truth"*)
- `neomjs/neo#17283` (`"no container-side code path attempts a GitHub read that cannot succeed"`)
- `neomjs/neo-agent-brain#583` (`"The Memory Core operation get_graph_scene (read tier) serves that read. fleetGraphSceneSource reads the scene through that one operation... The default budget holds today's graph whole"`)

## The Problem

1. **Design Amendment on `explore_lane_landscape` (`PR #1285` review `pullrequestreview-5647022042` `RA-1` by `@neo-opus-grace`):**
   - Initially this ticket proposed wiring `explore_lane_landscape` in `ai/mcp/server/memory-core/toolService.mjs:184-211` to `AiConfig.orchestrator.corpusProjection.materializedRoot`.
   - As surfaced by `@neo-opus-grace` against `laneLandscapeCensusSource.mjs:10` and `openWorkCensusReader.mjs` (`neomjs/neo#15234` / `neomjs/neo#15264`), a current-state open-work census **cannot** be answered from a local markdown projection because the projection lags live claims and carries neither issue nor pull-request assignee truth. `makeRefusingCensusPageReader('memory-core')` already satisfies `neomjs/neo#17283` by reporting honest `coverage.degraded: true` on the container plane rather than presenting a lagging snapshot as an exhausted census. The materialized-census readers are therefore dropped from scope.
2. **`explore_memory_history` Timed Out at `64,580 ms` (`errorCount: 1 / 1` in `get_memory_core_tool_metrics`):**
   - Live `get_memory_core_tool_metrics` recorded `callId: c47b8d32-7632-432a-8b40-c2f7e6b3d1ea` failing after `64,580.66 ms` (exceeding the 60s MCP client deadline).
   - Tracing `ai/mcp/server/memory-core/toolService.mjs:221-247` and `ai/services/memory-core/helpers/exploreMemoryHistory.mjs:65-93` reveals compounding bottlenecks:
     - `enumerateChronologicalWindowSources` (`ai/services/memory-core/helpers/chronologicalWindowSources.mjs:128`) walks `queryRecentTurns` **sequentially** across all rostered identities (`for (const identity of identityList)`), snapshotting the WAL and querying SQLite/Chroma one identity at a time.
     - `enumerateChronologicalWindowSources` and `fetchWindowSessions` run sequentially instead of in parallel.
     - After Chroma vector enrichment (`enrich`), `generateNarrative` (`ai/services/memory-core/helpers/temporalSynthesis.mjs:59`) invokes a synchronous LLM completion (`SessionService.model.generateContent`) with no bounded timeout fallback before the 60s MCP request deadline expires.
3. **`get_graph_scene` (`openapi.yaml:2243-2251`, `GraphService.mjs:1500`) Needs an Explicit Agent-Facing Description Redirect:**
   - Per `#583`, `get_graph_scene` must remain in `x-neo-tool-tier: read` with `maxNodes = 250000, maxEdges = 500000` because Fleet Manager's `fleetGraphSceneSource` calls it with `{}` to render the full Observatory graph (`avgDurationMs: 25,251 ms`).
   - However, because its `openapi.yaml` description says *"Returns a bounded node/edge scene from the Native Edge Graph for visual exploration"*, interactive agent sessions mistake it for a neighborhood graph query tool instead of calling `query_hybrid_graph`, `get_neighbors`, `get_node`, or `get_context_frontier`.

## The Architectural Reality

- `ai/services/memory-core/helpers/chronologicalWindowSources.mjs` (`lines 128-147`): per-identity `walkSource` loop for `partition: 'unified'`.
- `ai/services/memory-core/helpers/exploreMemoryHistory.mjs` (`lines 24-51, 96-125`) & `temporalBirdViewSynthesizer.mjs` (`lines 127-138`): chronological turn/session window walk and narrative synthesis seam.
- `ai/mcp/server/memory-core/openapi.yaml` (`lines 2243-2251`): `get_graph_scene` tool description.

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-25T22:46Z`): `ai/mcp/server/memory-core` and `ai/services/memory-core/helpers` own these surfaces; no new folders required.

## The Fix

1. **Keep `explore_lane_landscape`'s refusing plane reader per `neomjs/neo#15234` (`PR #15264`):**
   - Retain `makeRefusingCensusPageReader(censusUnavailableOnThisPlane)` in `toolService.mjs` so the plane honestly reports `coverage.degraded: true` rather than claiming census exhaustion from a lagging projection without PR assignee truth.
2. **Bound `explore_memory_history` latency (`chronologicalWindowSources.mjs`, `exploreMemoryHistory.mjs`):**
   - Parallelize the per-identity `walkSource` calls in `enumerateChronologicalWindowSources` (`Promise.all` over `identityList`, preserving deterministic sort order afterward) and run `enumerateChronologicalWindowSources` + `fetchWindowSessions` concurrently via `Promise.all`.
   - Bound `generateNarrative` with a strict timeout (`withTimeout(..., synthesisTimeoutMs)`, default `15000` ms) so if LLM synthesis exceeds budget, `synthesizeTemporalBirdView` catches the `timeout` error and returns the canonical degraded synthesis envelope (`synthesisAvailable: false`, `coverage.degraded: true`, `coverage.degradedReason: 'synthesis-failed: timeout'`) instead of timing out the entire MCP call at 60s.
3. **Clarify `get_graph_scene` in `openapi.yaml` (`lines 2243-2251`):**
   - Keep `get_graph_scene` in `x-neo-tool-tier: read` with its `#583` `250k/500k` defaults untouched so `fleetGraphSceneSource` stays green, and add a clear `IMPORTANT:` call-site steering note in `openapi.yaml` directing agent sessions to `query_hybrid_graph`, `get_neighbors`, `get_node`, and `get_context_frontier` for interactive graph queries.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `explore_memory_history` (`chronologicalWindowSources.mjs:128-147`, `exploreMemoryHistory.mjs:24-51, 96-125`) | Discussion `#20111` Option F | Walks per-identity turns in parallel (`Promise.all` with deterministic ordering), runs turn + session window reads concurrently, and bounds `generateNarrative` with `synthesisTimeoutMs` (default `15000` ms) | `synthesisAvailable: false`, `coverage.degraded: true`, `coverage.degradedReason: 'synthesis-failed: timeout'` when LLM exceeds budget | `exploreMemoryHistory.mjs` JSDoc | Unit tests in `chronologicalWindowSources.spec.mjs` and `exploreMemoryHistory.spec.mjs` + post-merge L3 live plane timing check (`#1290`) |
| `get_graph_scene` (`ai/mcp/server/memory-core/openapi.yaml:2243-2251`) | `#583` | Preserves `read` tier and `250k/500k` defaults for Fleet Observatory (`fleetGraphSceneSource`), while documenting in `openapi.yaml` that interactive agent queries should use `query_hybrid_graph` / `get_neighbors` / `get_node` / `get_context_frontier` | Unchanged | `openapi.yaml` | `McpServerToolLimits.spec.mjs` + `OpenApiValidatorCompliance.spec.mjs` |

## Decision Record impact

`aligned-with` `neomjs/neo#15234` (`PR #15264` — census cannot be answered from a local projection), `neomjs/neo#17283`, and `#583`.

## Acceptance Criteria

- [~] ~~`explore_lane_landscape` reads open issues and pull requests from `AiConfig.orchestrator.corpusProjection.materializedRoot` when present on the plane, returning non-degraded census coverage without attempting a container-side GitHub API call (`#17283`).~~ Retracted per `PR #1285` review `RA-1` (`@neo-opus-grace`): `neomjs/neo#15234` / `neomjs/neo#15264` explicitly rule out answering a current-state census from a local projection that lags live claims and lacks PR assignee truth; `makeRefusingCensusPageReader('memory-core')` remains the intended plane behavior.
- [ ] `enumerateChronologicalWindowSources` walks `identityList` in parallel while preserving deterministic output order, and `explore_memory_history` bounds LLM narrative generation (`synthesisTimeoutMs`) so a slow model call returns `synthesisAvailable: false`, `coverage.degraded: true`, `coverage.degradedReason: 'synthesis-failed: timeout'` with the structured window/citations rather than exceeding the 60s MCP deadline. `[L3-deferred — operator handoff needed: live plane <60s verification after deploy owned by #1290]`
- [ ] `get_graph_scene` in `openapi.yaml` keeps its `#583` `read`-tier and `250k/500k` defaults while explicitly directing agent sessions to `query_hybrid_graph`, `get_neighbors`, `get_node`, and `get_context_frontier`.
- [ ] Unit tests in `test/playwright/unit/` verify the parallel identity walk ordering and the bounded `explore_memory_history` synthesis timeout fallback.

## Out of Scope

- Changing `explore_lane_landscape`'s plane-side refusing reader (`neomjs/neo#15234` / `neomjs/neo#15264`).
- Changing `get_graph_scene`'s default `maxNodes`/`maxEdges` or `x-neo-tool-tier: read` (`#583` Design Authority requires those for `fleetGraphSceneSource`).
- Wiring a GitHub PAT into `mc-server` (`#17283` ruling).

## Avoided Traps

- **Answering a current-state census from a lagging markdown projection (`neomjs/neo#15234`):** The synced corpus projection does not carry PR assignee truth and lags live claims; presenting it with `exhausted: true` violates `laneLandscapeCensusSource.mjs:10`.
- **Moving `get_graph_scene` out of `read` tier or lowering its default `maxNodes: 250000`:** Breaks Fleet Manager's `fleetGraphSceneSource` (`#583`), which calls `get_graph_scene({})` on the read-tier plane client to render the Observatory graph.
- **Letting a slow LLM synthesis call abort the entire `explore_memory_history` window census:** The chronological window spine and Chroma citations are already computed before `generateNarrative` runs; throwing away the deterministic retrieval because the final prose synthesis timed out wastes the entire call.

## Related

- `neomjs/neo` Discussion `#20111` (Option F — comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])
- `neomjs/neo#15234`, `neomjs/neo#15264`, `neomjs/neo#17283`, `#583`, `#1280`, `#1290`

unowned-rationale: Claimed and implemented by `@neo-gemini-maryam` in `PR #1285`.

Live latest-open sweep: checked latest 25 open issues in `neomjs/neo-agent-brain` at `2026-10-25T22:48Z`; no equivalent open ticket found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-25T22:48Z`; no competing `[lane-claim]`.
MC sweep: `query_raw_memories` on `explore_lane_landscape materializedRoot explore_memory_history timeout`, 6 results, no prior decision declining this fix.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "explore_memory_history timeout chronologicalWindowSources get_graph_scene 15234"


## Timeline

- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `bug` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `performance` label
- 2026-10-25T22:52:40Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-25T22:52:41Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-26T01:21:34Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-26T01:41:59Z @neo-gemini-maryam cross-referenced by PR #1285
- 2026-10-26T02:11:06Z @neo-gemini-maryam cross-referenced by PR #1288
- 2026-10-26T03:34:56Z @neo-gemini-maryam referenced in commit `bce5d42` - "fix(memory-core): drop materialized census readers per neo#15234 design (#1281)"
- 2026-10-26T12:27:26Z @tobiu referenced in commit `c3145e7` - "fix(memory-core): bound explore_memory_history latency and clarify get_graph_scene (#1281) (#1285)

* fix(memory-core): wire explore_lane_landscape to materializedRoot and bound explore_memory_history (#1281)

* fix(memory-core): drop materialized census readers per neo#15234 design (#1281)"
- 2026-10-26T12:27:26Z @tobiu closed this issue

