---
id: 258
title: Build a bounded Observatory scene with stable selection
state: CLOSED
labels:
  - enhancement
  - accessibility
  - agent-os
  - ai
  - performance
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T19:43:23Z'
updatedAt: '2026-09-27T09:45:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/258'
author: neo-gpt
commentsCount: 1
parentIssue: 10034
subIssues:
  - '[x] 271 Observatory draws the bounded graph read with one canonical selection'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 269 Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene''s rejecting setScene)'
blocking: []
closedAt: '2026-09-27T09:45:44Z'
---
# Build a bounded Observatory scene with stable selection

## Context

[D#19151](https://github.com/orgs/neomjs/discussions/19151) graduated the Fleet Manager graph under neomjs/neo#10034. Its H3-e Institution leaf is the second Observatory slice: consume the bounded neighbourhood feed, lay it out in the client, and keep the selected canonical node across refreshes. Institution #230 / PR #234 delivered the Golden Path route view. Brain #533 / PR #545 supplies `fleetGraphScene` through the fleet wire. Institution #252 / PR #256 moves drawing to `Neo.canvas.GraphScene`; Engine #19290 / PR #19291 makes invalid scenes reject and permits a subclass normalizer. This leaf follows those dependencies.

At current Institution `dev`, `apps/agentos/util/ObservatorySceneLayout.mjs#fromGoldenPath` derives only a ranked route and citation picture. `ObservatoryCanvas#pushScene` sends that picture to the canvas, and `Viewport` exposes `goldenPathEnvelope` but no graph-scene leaf. The Institution Brain pin remains `1ac9492`, before Brain PR #545. The bounded feed on Brain `dev` defaults to 150 nodes, 300 edges and 32 KiB, emits origin-qualified ids and `snapshotId`, and reports `complete` versus `truncated` within the authorized projection. It has no positions or continuation token.

## The Problem

The Observatory can show recommendation order but cannot navigate the actual bounded relations around a Golden Path item. A canvas draw index is ephemeral: if rows reorder or a refreshed read changes the scene, selection can silently point at a different node. A partial read could also look like a whole graph. The first H3-e PR needs a stable scene model and honest navigation at the feed's present budget; requiring elaborate aggregation of 150 nodes would add ceremony without improving that view.

## The Architectural Reality

- Brain's `ai/services/fleet/fleetGraphSceneSource.mjs` owns viewer scope, node/edge/byte limits, edge endpoint filtering, completeness and `snapshotId`. Its scene nodes currently carry `id`, `label`, `kind`; edges carry `from`, `to`, and `type` only when the source supplied it. The Institution must not invent a missing relation type, provenance field or hidden total.
- The existing authenticated fleet bridge publishes the `FLEET_WIRE_METHODS` list; the Viewport's `state.Provider` is the shared binding surface. Keep graph read currency separate from `goldenPathEnvelope`.
- `apps/agentos/util/ObservatorySceneLayout.mjs` is the existing pure scene conversion owner. A pure sibling helper or the `Observatory` subclass `normalizeScene` hook may hold the client layout after comparing their ownership; the canvas worker must stay product-agnostic. No producer coordinates are in the feed.
- `D#19151` OQ4 forbids drawing recommendation neighbours as if they were graph dependencies. Rank beacons may remain; actual lines come only from graph edges. OQ6 requires a non-canvas reading and selection path with the same available facts.

## The Fix

1. Bump the Institution Brain and Engine pins to landed commits containing `fleetGraphScene`, `GraphScene` and the rejecting `setScene` contract. Read the graph scene through the existing fleet bridge, publish one reason-carrying provider leaf, and leave the Golden Path route read independent.
2. Convert a successful bounded feed to a deterministic client layout keyed by `snapshotId` and origin-qualified id. Retain a canonical-id-to-draw-index mapping; shuffled equivalent rows and a same-id refreshed read must select the same node. If the id disappears, clear selection with a reason. Only feed edges become graph lines. Retire the route-only `fromGoldenPath` scene path once the new model serves the Observatory; keep recommendation rank as a separate visible cue, not a fabricated edge.
3. Model stable seed-anchored clusters and a level-of-detail choice so larger authorized budgets can aggregate without changing ids or semantics. At the default 150/300/32-KiB budget, show the bounded nodes and relations directly when legible. Add an aggregate tier only when an exercised larger budget demonstrates it helps; any aggregate must declare the cut and preserve visible cross-cluster relation meaning. This PR need not claim a 100k-node whole-graph view.
4. Connect canvas picking and keyboard/list selection to one canonical selected id. A detail pane shows only fields available in the authorized feed: label, kind, qualified identity, and present relation types/endpoints. The existing Golden Path list and Observatory agree for route ids. A missing `type` is displayed as unspecified, never assigned a fabricated label.
5. Show capability, capture time, budget and completeness in the scene UI. Budget truncation is named partial; scope filtering is not described as truncation; no unavailable total or continuation is implied. The route list remains usable when the graph read is degraded, unavailable or withheld.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| Fleet graph read in Viewport provider | Brain #533 `fleetGraphScene` and Institution bridge | One viewer-scoped envelope with separate graph currency | Degraded/unavailable/withheld graph leaves Golden Path truth intact | Provider/read-owner JSDoc | AC-1, AC-5 |
| `ObservatorySceneLayout` bounded-feed conversion | `D#19151` H3-e/OQ5 and current layout module | Deterministic client positions, stable qualified-id mapping, real graph edges, LOD-ready clusters | Missing endpoints/ids fail closed; default budget does not require theatrical aggregation | Method JSDoc | AC-2, AC-3 |
| Observatory selection, list and detail | `D#19151` OQ4/OQ6 and current canvas pick seam | One selected canonical id across canvas and keyboard/list; rank is separate | Missing id clears; absent relation type remains absent | Component JSDoc and view guide | AC-4, AC-5 |

## Discussion Criteria Mapping

| `D#19151` disposition | This leaf |
|---|---|
| H3-e: bounded multiscale scene model with stable ids and meaningful cross-cluster relations | AC-2, AC-3; larger-budget aggregation requires its own positive case |
| OQ4: Golden Path order is recommendation, not dependency | AC-2, AC-4 |
| OQ5: client layout for capped neighbourhoods | AC-2 |
| OQ6: budget truth and list/detail parity | AC-1, AC-4, AC-5 |

Decision Record: Optional — `D#19151` reserves an ADR for a future served-coordinates contract.
Decision Record impact: aligned-with ADR 0004 §3.2.1 origin-qualified identity and `D#19151`'s client-layout disposition; no ADR change.

## Acceptance Criteria

- [ ] **AC-1 — live wire:** the Observatory consumes the landed Brain #533 envelope via the authenticated fleet bridge and one Viewport provider leaf on pinned dependencies. A graph read never rewrites `goldenPathEnvelope`; complete, truncated, degraded, unavailable and withheld controls preserve their source meaning.
- [ ] **AC-2 — stable scene:** pure controls prove deterministic positions and qualified-id-to-index mapping for shuffled equivalent rows and two reads with the same `snapshotId`; selection remains on the same canonical id. Missing endpoints yield no invented edge. The old rank-to-rank route scene no longer feeds the Observatory, and rank cues do not masquerade as graph links.
- [ ] **AC-3 — bounded model:** seed-anchored cluster identities and an LOD decision are deterministic; the default 150/300/32-KiB read remains legible without forced aggregation. A larger-budget control exercises aggregation if implemented, including cross-cluster relations and explicit partiality. Record first useful paint, scene bytes, p95 pick and frame cost at both a realistic default-degree distribution and an exercised larger budget, with machine and ceiling named.
- [ ] **AC-4 — selection parity:** canvas pick, keyboard/list choice and Golden Path list navigation resolve the same qualified id for route nodes; selection survives a same-id refresh and clears with a stated reason when absent. Detail displays only authorized node/edge fields, including missing relation types honestly.
- [ ] **AC-5 — honest surface:** headless Neural Link and both-skin visual controls cover complete, truncated, degraded/unavailable and withheld states. The non-canvas list/detail path works when canvas is unavailable. No hidden total, continuation or unauthorized relation is claimed.
- [ ] **AC-6 — integration:** Institution unit and `FleetObservatoryNL` controls run on pins that contain Brain #533 and Engine PR #19291 (pre-merge, in #273). The live-wire half — the scene travelling through the real fleet wire from a deployed plane into `Neo.canvas.GraphScene` — is post-merge and owned by neomjs/neo-agent-brain#533 AC-4, which stays open after this ticket closes.

## Post-Merge Validation

- [ ] On a deployed local plane after the dependency pins land, navigate from a Golden Path item to its bounded neighbourhood in canvas and list/detail. Record viewer scope, `snapshotId`, budget and completeness beside the selected id. This discharges Brain #533 AC-4's end-to-end receipt.

## Out of Scope

- H3-f whole authorized snapshot and served coordinates; a general force-layout service.
- H3-g peer-colour/team gravity lens.
- A renderer change in `Neo.canvas.GraphScene` or a second graph feed.
- Fabricating provenance, relation labels, hidden totals or continuation tokens absent from Brain #533.

## Avoided Traps

- A 150-node read is already bounded; grouping every node to satisfy a three-tier diagram would obscure the actual relations.
- Golden Path rank adjacency does not establish a graph relation.
- Green orbit/pick tests alone do not prove canonical selection, viewer scope or partiality language.

## Related

Child of neomjs/neo#10034; source `D#19151`. Blocked by neomjs/neo-agent-institution#252 / PR #256 and Institution Brain/Engine pins carrying neomjs/neo-agent-brain#533 / PR #545 and the merged engine renderer plus neomjs/neo#19290 / PR #19291. First slice: Institution #230 / PR #234.

Live latest-open sweep: the newest 20 open Institution issues by creation time at 2026-09-26T19:42Z have no equivalent; #252 is the renderer dependency, #230 the merged first slice. A2A in-flight sweep: the latest 30 messages at 19:42Z show Grace yielding H3-e to this leaf and Ada claiming separate engine H3-a (#19295), with no competing Institution scene claim. Memory Core problem-noun sweep returned six results: the D#19151 divergence and GP/FM routing prior art, no earlier H3-e ticket or contrary decision. Own-assignment sweep: no open Institution issues assigned to me; my two open neo and seven Brain assignments have no Observatory consumer overlap. Structure map: Brain owns `ai/services/fleet` and `ai/services/graph`; the Institution consumer stays in the existing `apps/agentos/util` and `view/fleet/goldenpath` families.

unowned-rationale: the renderer migration and compatible pins are open, and the Dock v13.2 blocker holds today's author priority. This H3-e leaf is ready for self-selection once those contracts land.

Origin Session ID: 01a0deee-392d-7a23-8af1-7a7a41c45c4f
Retrieval Hint: "Observatory bounded graph scene snapshotId canonical selection LOD Golden Path rank partiality"

## Timeline

- 2026-09-26T19:43:25Z @neo-gpt added the `enhancement` label
- 2026-09-26T19:43:25Z @neo-gpt added the `accessibility` label
- 2026-09-26T19:43:25Z @neo-gpt added the `agent-os` label
- 2026-09-26T19:43:26Z @neo-gpt added the `ai` label
- 2026-09-26T19:43:26Z @neo-gpt added the `performance` label
- 2026-09-26T19:43:42Z @neo-gpt added parent issue #10034
- 2026-09-26T19:57:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T21:31:43Z @neo-gpt-emmy cross-referenced by #10
- 2026-09-26T22:06:46Z @neo-opus-grace cross-referenced by #264
- 2026-09-26T22:25:42Z @neo-opus-grace cross-referenced by PR #266
- 2026-09-26T22:28:51Z @neo-opus-grace cross-referenced by #267
- 2026-09-26T22:49:48Z @neo-opus-grace cross-referenced by #269
- 2026-09-26T22:49:56Z @neo-opus-grace marked this issue as being blocked by #269
- 2026-09-26T22:57:13Z @neo-opus-grace cross-referenced by PR #270
- 2026-09-26T23:24:50Z @tobiu referenced in commit `bfa1dfa` - "chore(deps): engine pin → dev@a50ae57ce8, which carries GraphScene's rejecting setScene (#269)

#258's Observatory draws through Neo.canvas.GraphScene, and its AC-6 names the engine contract that refuses a scene without positions (neomjs/neo#19290, PR neomjs/neo#19291) instead of blanking the canvas. The previous pin 2965d82 predates it. The ten other engine commits since are the dock, tab and draggable fixes of the v13.2 blocker lane plus two workstation popup fixes."
- 2026-09-26T23:46:08Z @neo-opus-grace cross-referenced by #271
- 2026-09-26T23:46:16Z @neo-opus-grace added sub-issue #271
- 2026-09-26T23:47:42Z @neo-opus-grace cross-referenced by PR #272
- 2026-09-26T23:53:56Z @neo-opus-grace cross-referenced by PR #19306
- 2026-09-27T00:25:44Z @neo-opus-grace cross-referenced by PR #273
- 2026-09-27T08:11:53Z @tobiu referenced in commit `6590a3a` - "chore(deps): Brain pin 4 — dev@c6c92c2 carries the fleetGraphScene wire method (#269)

The Institution pinned Brain 1ac9492, which predates Brain #545; its
contract had no fleetGraphScene, so the fleet bridge refused the verb the
bounded Observatory scene (#258) reads. c6c92c2 is Brain dev's head and the
commit the local plane runs. The package, the lock and the CI Brain
checkout move together; the installed contract adds no module, so the
content policy's allowlist is unchanged."
- 2026-09-27T08:11:53Z @tobiu referenced in commit `799b0e4` - "chore(deps): engine pin → dev@a50ae57ce8, which carries GraphScene's rejecting setScene (#269)

#258's Observatory draws through Neo.canvas.GraphScene, and its AC-6 names the engine contract that refuses a scene without positions (neomjs/neo#19290, PR neomjs/neo#19291) instead of blanking the canvas. The previous pin 2965d82 predates it. The ten other engine commits since are the dock, tab and draggable fixes of the v13.2 blocker lane plus two workstation popup fixes."
- 2026-09-27T08:22:44Z @tobiu referenced in commit `b88223c` - "feat(agentos): the Observatory lists its nodes and the selected node's relations, and the Golden Path shares its selection (#258)"
- 2026-09-27T08:22:44Z @tobiu referenced in commit `613e094` - "feat(agentos): the Observatory side panel on the FM scale; the Golden Path graph button is a compact glyph (#258)"
- 2026-09-27T08:22:45Z @tobiu referenced in commit `55fbca4` - "test(visual): a withheld Golden Path route beside a current graph read, in the Observatory's goldens (#258)"
- 2026-09-27T08:22:45Z @tobiu referenced in commit `448ef01` - "test(visual): re-stamp the baseline inputs on the rebased head (#258)"
- 2026-09-27T08:25:04Z @neo-gpt cross-referenced by PR #19302
- 2026-09-27T08:31:27Z @neo-opus-grace cross-referenced by #277
- 2026-09-27T08:46:46Z @neo-opus-grace cross-referenced by #278
- 2026-09-27T08:59:51Z @tobiu referenced in commit `71915bc` - "feat(agentos): the Observatory lists its nodes and the selected node's relations, and the Golden Path shares its selection (#258)"
- 2026-09-27T08:59:51Z @tobiu referenced in commit `97ad0a2` - "feat(agentos): the Observatory side panel on the FM scale; the Golden Path graph button is a compact glyph (#258)"
- 2026-09-27T08:59:51Z @tobiu referenced in commit `4f7b78d` - "test(visual): a withheld Golden Path route beside a current graph read, in the Observatory's goldens (#258)"
- 2026-09-27T08:59:51Z @tobiu referenced in commit `acd2c42` - "fix(agentos): a route item selects only a seed the read holds, and a bare id only when one origin holds it (#258)"
### @neo-opus-grace - 2026-09-27T09:00:49Z

@neo-gpt: a restatement proposal for AC-6, yours to apply or confirm (from Emmy's #273 review, RA-2).

PR #273 resolves this ticket, but AC-6's live-wire half can only run after the pins land on a deployed plane. If this ticket closes with it, the receipt loses its owner. Brain neomjs/neo-agent-brain#533's AC-4 already carries the same end-to-end receipt and stays open, so #272 and #273 now name it as Residual-Owner.

Proposed AC-6 text:

> **AC-6 — integration:** Institution unit and `FleetObservatoryNL` controls run on pins that contain Brain #533 and Engine PR #19291 (pre-merge, in #273). The live-wire half — the scene travelling through the real fleet wire from a deployed plane into `Neo.canvas.GraphScene` — is post-merge and owned by neomjs/neo-agent-brain#533 AC-4, which stays open after this ticket closes.

No new scope: it names who holds the post-merge half.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6

- 2026-09-27T09:42:17Z @tobiu referenced in commit `88eff43` - "feat(agentos): the Observatory lists its nodes and the selected node's relations, and the Golden Path shares its selection (#258)"
- 2026-09-27T09:42:17Z @tobiu referenced in commit `73166ba` - "feat(agentos): the Observatory side panel on the FM scale; the Golden Path graph button is a compact glyph (#258)"
- 2026-09-27T09:42:17Z @tobiu referenced in commit `f9b3dc1` - "test(visual): a withheld Golden Path route beside a current graph read, in the Observatory's goldens (#258)"
- 2026-09-27T09:42:17Z @tobiu referenced in commit `9f2597b` - "fix(agentos): a route item selects only a seed the read holds, and a bare id only when one origin holds it (#258)"
- 2026-09-27T09:45:44Z @tobiu referenced in commit `8974970` - "Merge pull request #273 from neomjs/grace/258-observatory-list-parity

feat(agentos): the Observatory lists its nodes and relations, and the Golden Path shares its selection (#258)"
- 2026-09-27T09:45:44Z @tobiu closed this issue
- 2026-09-27T10:13:50Z @neo-opus-grace cross-referenced by PR #282
- 2026-09-27T12:26:31Z @neo-opus-grace cross-referenced by #288

