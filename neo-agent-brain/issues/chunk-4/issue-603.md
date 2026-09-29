---
id: 603
title: The graph scene carries the Brain's gravity and recency columns
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T11:42:14Z'
updatedAt: '2026-09-29T11:31:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/603'
author: neo-opus-vega
commentsCount: 0
parentIssue: 312
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-29T11:31:05Z'
---
# The graph scene carries the Brain's gravity and recency columns

## Context

Stage B1 of D#19317, the Observatory's product definition, which graduated at body `updatedAt 2026-09-28T10:56:25Z` (Signal Ledger below). After Stage C, the Observatory's default geography is W3: wells at the Brain's own strategic anchors. On two viewers' live scenes those anchors read like the roadmap (`Fleet Manager (FM)`, `ADR-0019`, `Golden Path`, `Workstation`, `Neural Link`, `Dock Layouts Epic`). None of it leaves the Brain today.

## The Problem

`GraphService#readSceneGraph` answers each node with `id`, `kind` and `label`, "never its property bag". The signals W3 and the heat overlay need stay behind:

| Signal | Stored as | Live coverage (2026-09-28) | Use |
|---|---|---|---|
| strategic anchor | `properties.gravity_well` (REM Tri-Vector, origin `#9786`) | ≥ 8,619 nodes | W3 wells (Q1) |
| anchor mass | `properties.strategic_weight` | 21,873 nodes ≥ 0.5 | W3 ranking |
| last activity | issue/PR `updatedAt`, file/directory `mtimeMs`, agent memory `timestamp`, retrospective `discoveredAt` | 71–100 % per kind; concepts, classes and sessions have none | heat overlay (Q2) |

## The Architectural Reality

- `ai/services/memory-core/GraphService.mjs` `readSceneGraph`: keyset pages of 5,000 rows under the SQL RLS clause, each row's `data` parsed, columnar answer `{kinds, types, nodes: {ids, kinds, labels}, edges, counts, budget, truncated}`.
- `ai/mcp/server/memory-core/openapi.yaml`: the `get_graph_scene` output schema.
- `ai/services/fleet/fleetGraphSceneSource.mjs` `projectScene`: maps the answer to `{id, label, kind}` nodes within a 64 MiB byte budget.
- Measured in an isolated probe container (2026-09-28, viewer `neo-opus-ada`): the whole read costs 1.6 s and about 320 MB (166,302 nodes, 238,612 edge rows, 82 page yields). The installed FM's first two reads took 111.6 s and 65.0 s against the client's 60 s timeout. The cause is named under AC-1.
- The store keeps no capture time for any source: the Nodes table has no write or sync column, and no writer stamps one into a node's properties.

## The Fix

Named columns aligned with `nodes.ids`, never the property bag (D#19317 OQ-W3):
- `nodes.gravityWell`: 0/1.
- `nodes.strategicWeight`: a number or `null`.
- `nodes.lastActivityAt`: epoch ms or `null`, from the per-kind map above. The answer names each kind's field once, in a top-level `activitySources` map `{KIND: {field, sourceCapturedAt}}`, instead of a per-node code. `sourceCapturedAt` is `null` because the store keeps no capture time, which is D#19317 B1's "or explicitly unknown". A kind absent from the map has no source (STEP_BACK point 4).

`projectScene` forwards the columns and the map; a scene without them projects as today, so a consumer on an older Brain stays on W2. The Fleet envelope's `capturedAt` is the read's time and never stands in for a source's.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_graph_scene` output | `GraphService#readSceneGraph` | + `nodes.gravityWell`, `nodes.strategicWeight`, `nodes.lastActivityAt`, and top-level `activitySources` `{KIND: {field, sourceCapturedAt: null}}` | absent property → `0` / `null` | `openapi.yaml` | unit arms on a fixture store |
| fleet `fleetGraphScene` | `projectScene` | + node `gravityWell`, `strategicWeight`, `lastActivityAt`; the scene forwards `activitySources` as the Brain states it | absent column → field omitted | module JSDoc | projection spec, with a fresh read over an old and an unknown source capture |

## Acceptance Criteria

- [x] AC-1 The cold `get_graph_scene` read's cause is named before this leaf adds bytes (D#19317 §7; Emmy: investigate before changing the timeout or the payload).
  - **Named (2026-09-28, Memory Core's own `mc_tool_call_log`):** the read was slow because Memory Core as a whole was stalled, not because of the read's own cost.
    - On a server that is not stalled the read takes 2.2–2.9 s (08:16, 08:42, and 10:25:11 with the full 18.8 MiB answer).
    - The installed FM launched cold at 10:19:55 and sent six reads in the same second: the scene, `get_rem_pipeline_state` twice, `get_computed_route` twice, and the deployment snapshot. An identical set followed at 10:21:54–10:22:02. In that 15-minute window Memory Core, running on 1 CPU, carried 273 calls instead of its usual ~100, 174 of them a `list_messages` drain.
    - Healthchecks queued for up to 177.9 s. `query_recent_turns` took 114.6 s and `get_rem_pipeline_state` 35.8 s, against its p50 of 0.43 s.
  - **Not named: what holds the loop.** Memory Core stalls for 10 s or more several times in every 15 minutes, all day, and the stalls do not grow with process age (71 s at 13:00, after the 11:10 restart). The six reads normally cost about 8 s together. Defect-note, 2026-09-28.
  - **Consequence for this leaf:** 5.6% more bytes on a 2–3 s read does not move a stall measured in minutes, and a longer timeout would only hide the stall.
- [ ] AC-2 `readSceneGraph` returns the columns aligned with `nodes.ids`; an RLS-hidden node contributes nothing.
- [ ] AC-3 `lastActivityAt` follows the per-kind source map. `activitySources` names each kind's field and states its capture time as unknown (`sourceCapturedAt: null`), and a kind absent from the map has no source.
- [ ] AC-4 `projectScene` forwards the columns; a scene without them projects as today.
- [ ] AC-5 `openapi.yaml` documents the columns; the tool's output-schema validation passes.
- [ ] AC-6 Byte and time cost measured before and after on the live plane and recorded in the PR (+5.6 % expected for the node columns on the 18.33 MiB answer).
- [ ] AC-7 It measures the issues still missing `author` and re-projects where the source has one (STEP_BACK point 6).
- [ ] AC-8 Unit arms go red on `dev`.

## Out of Scope

- Attribution and origin (B2); the consumers (Institution Stages A and C).
- Edge weight (only if W4 earns an accepted AC); heat propagation onto concepts.
- A per-source capture time: none is stored today, so the leaf states it unknown instead of inventing one.

## Avoided Traps

- **Passing the property bag through:** it would leak fields past RLS intent and bloat the wire.
- **Free-form timestamps:** one normalized value per node, with a named source.
- **The read time as freshness:** a fresh scene read says nothing about when a source last synced.

Decision Record: NOT_NEEDED (D#19317)
Decision Record impact: none

## Signal Ledger
- `claude`: AUTHOR_SIGNAL by @neo-opus-vega @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGdj`)
- `gpt`: APPROVED by @neo-gpt @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGjY`)

## Unresolved Dissent
(none at the final body anchor)

## Unresolved Liveness
- `unknown` family (@neo-preview, active): no graduation signal; not required for quorum.
- `gemini`, `kimi`: `operator_benched`, no signal.

## Discussion Criteria Mapping
- OQ-W3 (geometry as named columns) → AC-2
- OQ-W5 (cost) → AC-6; the cold read → AC-1
- STEP_BACK point 4 (event source, unknown state) → AC-3
- STEP_BACK point 6 (author re-projection, old-Brain fallback) → AC-4, AC-7

Related: D#19317 · neomjs/neo-agent-institution#10 · neomjs/neo-agent-institution#310 (Stage A) · `#442` · `#542` · `#9786`
Live latest-open sweep: latest 20 open Brain issues read at 11:32:37Z and the latest 8 at 11:39:59Z, plus an org-wide open-issue search for gravityWell and strategicWeight; no equivalent found.
MC sweep: "Observatory featureless sphere inbox wells mail dominates graph", 5 results, no prior decision found.
Own-assignment sweep: 9 open Brain assignments; #442 (the Graph consumer's corpus read) is adjacent but separate; none overlapping.
Origin Session ID: 96f97500-4dcb-461e-bef0-af4e6dc5e24a
Retrieval Hint: "graph scene columns gravity_well strategic_weight lastActivityAt readSceneGraph"


## Timeline

- 2026-09-28T11:42:15Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T11:42:17Z @neo-opus-vega added the `enhancement` label
- 2026-09-28T11:42:17Z @neo-opus-vega added the `ai` label
- 2026-09-28T11:42:17Z @neo-opus-vega added the `agent-os` label
- 2026-09-28T11:42:37Z @neo-opus-vega cross-referenced by #604
- 2026-09-28T11:42:59Z @neo-opus-vega cross-referenced by #311
- 2026-09-28T11:52:13Z @neo-opus-vega cross-referenced by #312
- 2026-09-28T12:05:56Z @neo-gpt-emmy added parent issue #312
- 2026-09-28T12:32:04Z @neo-opus-vega cross-referenced by PR #313
- 2026-09-28T13:12:12Z @neo-gpt-emmy cross-referenced by PR #605
- 2026-09-28T15:32:16Z @neo-opus-vega cross-referenced by PR #611
- 2026-09-28T18:22:34Z @neo-opus-vega referenced in commit `49a7117` - "fix(memory-core): the scene states each source's capture time as unknown (#603)

activitySources now names, per kind, the field lastActivityAt is read
from and sourceCapturedAt: null. The store keeps no capture time for
any source, so heat can use the event time but must treat each
source's freshness as unknown; the Fleet envelope's capturedAt is the
read's time and never stands in for it."
- 2026-09-29T09:28:01Z @neo-preview cross-referenced by PR #620
- 2026-09-29T09:44:22Z @neo-preview cross-referenced by #621
- 2026-09-29T11:31:05Z @tobiu referenced in commit `7f22b22` - "Merge pull request #611 from neomjs/vega/603-scene-gravity-recency

feat(memory-core): the graph scene carries gravity and recency columns (#603)"
- 2026-09-29T11:31:05Z @tobiu closed this issue
- 2026-09-29T11:48:23Z @neo-opus-vega cross-referenced by #320
- 2026-09-29T12:24:39Z @neo-preview cross-referenced by PR #321
- 2026-09-29T13:24:41Z @neo-opus-vega cross-referenced by #625
- 2026-09-29T13:36:28Z @neo-opus-vega cross-referenced by PR #626

