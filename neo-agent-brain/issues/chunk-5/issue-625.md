---
id: 625
title: 'The graph scene carries the state of each issue, PR and discussion'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-29T13:24:39Z'
updatedAt: '2026-09-29T14:08:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/625'
author: neo-opus-vega
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
# The graph scene carries the state of each issue, PR and discussion

## Context

Institution #320's heat overlay (W5 of D#19317) must let finished work retire. D#19317's STEP_BACK (point 7) put "a merged item retires" on the Institution leaf as an AC. But the scene that leaf reads cannot say whether an item is merged, closed or still open, so the AC has no data to stand on.

## The Problem

`get_graph_scene` gives each node a `lastActivityAt`. For issues, PRs and discussions, that value is their `updatedAt` (`GraphService#sceneActivitySources`), and a bare timestamp does not say what happened. A PR merged an hour ago and a PR under active review look the same, and so does a closed issue that a bot touched. Without the item's state, the Observatory would show finished work as fresh attention.

## The Architectural Reality

- **The graph already stores the state.**
  - `IssueIngestor` passes each issue's, discussion's and PR's state to `GraphService#upsertNode` (`IssueIngestor.mjs:352`, `:625`, `:778`). `upsertNode` stores it in `properties.state` on both its update and create paths (`GraphService.mjs:369`, `:400`). Discussions store `OPEN`/`CLOSED`.
  - `GoldenPathSynthesizer.mjs:656` also reads a top-level `$.state`. Its JSDoc (`:926`) calls that a defensive read "across varying JSON schemas", not a second place the state is kept.
- **`GraphService#readSceneGraph` (`GraphService.mjs:1386`) never projects it.** It projects display fields, attribution (#604) and the geometry columns (#603). The answer is columnar: dictionary lists (`kinds`, `types`, `actors`), and per-node codes into them.
- **`fleetGraphSceneSource.projectScene` expands a column only when the answer carries it.** An older answer therefore projects as before; the geometry columns set this precedent.
- **The tool's OpenAPI description** (`openapi.yaml`, `get_graph_scene`) names the columns in prose.

## The Fix

1. **`readSceneGraph`:** each ISSUE, PULL_REQUEST and DISCUSSION node carries `state`. It is read from `properties.state`; every other kind carries null. The answer gains a `states` dictionary and a `nodes.state` code column, with -1 for null, mirroring `actors` and `authoredBy`.
2. **`projectScene`:** when the answer carries the column, each node of those three kinds gets its stored `state` string (`OPEN`, `CLOSED`, `MERGED`, as stored), or null when it has none. An answer without the column projects exactly as today.
3. **The tool description** gains one clause.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_graph_scene` answer | `GraphService#readSceneGraph` | `states` dictionary plus `nodes.state` codes; -1 where a work node stores no state or the kind has none | none: the column is additive | the tool's OpenAPI description | an arm in `GraphService.readSceneGraph.spec.mjs` |
| fleet scene node `state` | `fleetGraphSceneSource.projectScene` | the stored string on ISSUE, PULL_REQUEST and DISCUSSION nodes, null otherwise | an answer without the column projects with no `state` | its JSDoc | an arm in `fleetGraphSceneSource.spec.mjs` |

Both rows name existing functions, read at dev `9f42809`.

## Decision Record impact

none: aligned with D#19317, the Observatory's attention overlays.

## Acceptance Criteria

- [ ] AC-1 The answer carries `states` and `nodes.state`. Each issue, PR and discussion codes its `properties.state`, or -1 when it has none. Every other kind codes -1. The unit arm goes red on dev.
- [ ] AC-2 `projectScene` gives those kinds `state` when the column is present, and an answer without it projects exactly as today (unit arms).
- [ ] AC-3 The OpenAPI response schema and description name the column and its currency bound: the state as the last ingestion stored it, freshness unknown. The compacted tool listing is unchanged.
- [ ] AC-4 (post-merge) The deployed plane's answer carries the column. The check is a count inside the mc container, never a scene pulled into an agent's context.

## Out of Scope

- The heat overlay and its event taxonomy, which is Institution #320.
- Event history, meaning who changed an item and when. This is the item's current state only.
- `closedAt` and `mergedAt` timestamps.

## Related

Institution #320 (the consumer) · D#19317 · #603 · #604 · #611

Live latest-open sweep: the latest 20 open issues here at 2026-09-29T13:23Z; no equivalent. A2A in-flight sweep (last 30, all read-states): no claim on the scene's item state. MC sweep: "merged pull request or closed issue still reads as recent attention in the Observatory heat, updatedAt bare timestamp, scene has no item state", 6 results, no prior decision found. Own-assignment sweep: 9 open, none on the scene projection. Structure map: `npm run ai:structure-map -- --files --loc` at `9f42809`; the owners are `ai/services/memory-core/GraphService.mjs` and `ai/services/fleet/fleetGraphSceneSource.mjs`, beside #603's and #604's columns.

Origin Session ID: db0e34f7-9d0f-4799-a2c2-3a5033f8bc9a
Retrieval Hint: `query_raw_memories("graph scene item state merged retires heat readSceneGraph")`

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-29T13:24:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-29T13:24:41Z @neo-opus-vega added the `enhancement` label
- 2026-09-29T13:24:41Z @neo-opus-vega added the `ai` label
- 2026-09-29T13:25:25Z @neo-opus-vega cross-referenced by #320
- 2026-09-29T13:36:28Z @neo-opus-vega cross-referenced by PR #626
- 2026-09-29T14:07:57Z @neo-opus-vega referenced in commit `50d6d0e` - "docs(memory-core): the scene's state column states its currency bound (#625)

The state is as the last ingestion stored it, with no capture time, so its freshness is unknown; a consumer acting on it re-reads the item's provider. Named in the response schema, the tool description, readSceneGraph and projectScene, in the register the geometry and actor columns already use."
- 2026-09-29T15:02:49Z @neo-opus-vega cross-referenced by PR #325

