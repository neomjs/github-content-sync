---
id: 271
title: Observatory draws the bounded graph read with one canonical selection
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T23:46:07Z'
updatedAt: '2026-09-27T09:45:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/271'
author: neo-opus-grace
commentsCount: 0
parentIssue: 258
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 269 Brain pin 4 (dev@c6c92c2, fleetGraphScene) + engine pin (dev@a50ae57ce8, GraphScene''s rejecting setScene)'
blocking: []
closedAt: '2026-09-27T09:45:00Z'
---
# Observatory draws the bounded graph read with one canonical selection

## Context

#258 (H3-e of `D#19151`) is larger than one PR: its six ACs span the read, the scene, canvas selection, a non-canvas list/detail path, Golden Path list parity, perf receipts and a live-wire receipt. The ticket-create label rule makes a standalone ticket one-PR-resolvable, so the first vertical slice gets its own leaf here. #258 stays open with all its ACs; the second slice resolves it.

## The Problem

At dev the Observatory binds `goldenPathEnvelope` and `ObservatorySceneLayout.fromGoldenPath` draws only the ranked route and its citations. Brain #533's `fleetGraphScene` (bounded neighbourhood, origin-qualified ids, `snapshotId`, `complete` vs `truncated`) has no reader, no leaf and no scene. A canvas draw index is ephemeral, so a selection keyed by it would silently land on another node after a refresh.

## The Architectural Reality

- The read path is `ReadingSurfacesController` (the Golden Path read's fence discipline) behind the authenticated fleet bridge, landing in the Viewport's `state.Provider`. `setData` drills objects into leaf paths, so a whole-envelope leaf must land in a closed shape (`GoldenPathEnvelope` already does; the projection lifts into a shared `ClosedShape`).
- `ObservatorySceneLayout` is the pure scene owner; `fromGraphScene` (already on the #258 branch) lays out seeds in their route slots with neighbours ringed around their nearest seed.
- The renderer is `AgentOS.canvas.Observatory` on `Neo.canvas.GraphScene`; `pick` answers the engine's index, which the subclass maps to its node. The engine pin needs neomjs/neo#19291's rejecting `setScene` (#269 / #270).

## The Fix

1. `loadGraphScene` beside `loadGoldenPath` (own fence, unavailable fallbacks with their own reasons) writes `graphSceneEnvelope`; the construction-time and reconnect paths re-drive it like the Golden Path read.
2. The Observatory binds that leaf, derives the scene once through `fromGraphScene`, retires `fromGoldenPath`, and shows the read's line (capability, capture, holdings, completeness or `partial, budget …`).
3. Selection is one origin-qualified id held by the pane: a click selects, an orbit does not, a new read that holds the id keeps it, a read that lost it clears it with the reason. A strip names the selection with only authorized fields: label, kind, qualified id, rank, and relations by type, with a missing type shown as `unspecified`.
4. The renderer inks by hop (the signal only while current), fades outside the selection's neighbourhood, draws only feed edges, and gains `locate`, the inverse of `pick`.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `graphSceneEnvelope` provider leaf | Brain #533 `fleetGraphScene`, fleet bridge | One closed envelope per read; never writes `goldenPathEnvelope` | Unwired/failed read lands `unavailable` with its reason | `GraphSceneEnvelope`, `ReadingSurfacesController` JSDoc | AC-1 |
| `ObservatorySceneLayout.fromGraphScene` | `D#19151` OQ4/OQ5 | Deterministic positions, id→index map, feed edges only | Absent endpoint drops the edge; cut seed leaves its slot | Method JSDoc | AC-2 |
| Observatory selection + `locate` | #258 AC-4 (canvas half) | One canonical id across refreshes | Missing id clears with a reason | Component JSDoc | AC-3 |

## Acceptance Criteria

- [ ] AC-1: the cockpit reads `fleetGraphScene` into `graphSceneEnvelope` without touching `goldenPathEnvelope`. Unwired, failed and stale-generation reads behave like the Golden Path read's.
- [ ] AC-2: `fromGraphScene` lays out shuffled rows, duplicated rows and two reads of one snapshot identically. It invents no edge and no rank link, and `fromGoldenPath` no longer feeds the Observatory.
- [ ] AC-3: a click on a drawn node selects its qualified id and an orbit selects nothing. A refreshed read that holds the id keeps it; one that lost it clears with the reason. The strip shows only feed fields, with untyped relations as `unspecified`.
- [ ] AC-4: NL controls cover complete, truncated, degraded-with-scene and unavailable on the pinned engine. Both-skin goldens cover the same states plus a selection.

## Out of Scope

- The non-canvas list/detail path, keyboard selection and Golden Path list parity; perf receipts; the withheld control (the producer reads no route admission yet); wiring per-node clusters into the engine's LOD (neomjs/neo#19302); the live-wire receipt. All stay on #258.

## Related

Sub of #258. Depends on #269 / #270 (pins). Precedent: #230 / #234, #252 / #256.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T23:45Z hold no slice ticket (#258 is the parent). A2A: the latest 30 messages hold no Observatory claim. Memory Core: no prior decision against slicing H3-e; #258's own sweep recorded Grace yielding H3-e to that leaf.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "Observatory graphSceneEnvelope fromGraphScene canonical selection locate slice 1"

## Timeline

- 2026-09-26T23:46:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T23:46:08Z @neo-opus-grace added the `enhancement` label
- 2026-09-26T23:46:08Z @neo-opus-grace added the `agent-os` label
- 2026-09-26T23:46:09Z @neo-opus-grace added the `ai` label
- 2026-09-26T23:46:16Z @neo-opus-grace added parent issue #258
- 2026-09-26T23:46:17Z @neo-opus-grace marked this issue as being blocked by #269
- 2026-09-26T23:47:42Z @neo-opus-grace cross-referenced by PR #272
- 2026-09-26T23:49:51Z @neo-opus-grace cross-referenced by #19305
- 2026-09-27T00:22:26Z @tobiu referenced in commit `51b42c3` - "test(visual): re-stamp the baseline inputs over the committed goldens (#271)"
- 2026-09-27T00:25:44Z @neo-opus-grace cross-referenced by PR #273
- 2026-09-27T08:19:41Z @tobiu referenced in commit `295d4f0` - "feat(agentos): lay out the bounded graph read around its seeds, stable per id (#271)

ObservatorySceneLayout.fromGraphScene turns a fleetGraphScene envelope into
a scene: the route's seeds on the helix in their route slots with their rank,
every other node ringed around the seeds nearest to it (hop, cluster), and
only the feed's own edges with their types. Positions depend only on ids,
route order and edges, so shuffled rows and a second read of one snapshot lay
out identically and the id-to-index map keeps a selection on its node."
- 2026-09-27T08:19:42Z @tobiu referenced in commit `77d2407` - "refactor(agentos): ClosedShape lands a wire envelope in its leaf's declared shape (#271)

The projection moves out of GoldenPathEnvelope unchanged, so the graph scene leaf lands through the same closed-shape rule: setData drills objects into leaf paths, so every declared key must be present on every write."
- 2026-09-27T08:19:42Z @tobiu referenced in commit `b485c9d` - "feat(agentos): the Observatory draws the Golden Path's bounded graph neighbourhood, with one canonical selection (#271)

The cockpit reads fleetGraphScene beside the Golden Path, on its own fence, into its own graphSceneEnvelope leaf. The Observatory binds that leaf, derives the scene once through ObservatorySceneLayout.fromGraphScene and retires the route-only fromGoldenPath scene: only the feed's edges become lines, and seeds are rank beacons. Selection is one origin-qualified id. A click selects, an orbit does not, and a read that lost the id clears the selection with its reason. The renderer inks by hop, fades outside a selection's neighbourhood and gains locate, the inverse of pick."
- 2026-09-27T08:19:42Z @tobiu referenced in commit `61b6a1c` - "test(visual): re-stamp the baseline inputs on the rebased head (#271)"
- 2026-09-27T08:31:27Z @neo-opus-grace cross-referenced by #277
- 2026-09-27T08:46:46Z @neo-opus-grace cross-referenced by #278
- 2026-09-27T08:55:44Z @tobiu referenced in commit `cb995d2` - "fix(agentos): an orbit never selects, a late pick cannot bring back an id, and a profile switch retires the graph (#271)"
- 2026-09-27T09:40:11Z @tobiu referenced in commit `7365ea7` - "feat(agentos): lay out the bounded graph read around its seeds, stable per id (#271)

ObservatorySceneLayout.fromGraphScene turns a fleetGraphScene envelope into
a scene: the route's seeds on the helix in their route slots with their rank,
every other node ringed around the seeds nearest to it (hop, cluster), and
only the feed's own edges with their types. Positions depend only on ids,
route order and edges, so shuffled rows and a second read of one snapshot lay
out identically and the id-to-index map keeps a selection on its node."
- 2026-09-27T09:40:11Z @tobiu referenced in commit `83fbc60` - "refactor(agentos): ClosedShape lands a wire envelope in its leaf's declared shape (#271)

The projection moves out of GoldenPathEnvelope unchanged, so the graph scene leaf lands through the same closed-shape rule: setData drills objects into leaf paths, so every declared key must be present on every write."
- 2026-09-27T09:40:11Z @tobiu referenced in commit `ae6effc` - "feat(agentos): the Observatory draws the Golden Path's bounded graph neighbourhood, with one canonical selection (#271)

The cockpit reads fleetGraphScene beside the Golden Path, on its own fence, into its own graphSceneEnvelope leaf. The Observatory binds that leaf, derives the scene once through ObservatorySceneLayout.fromGraphScene and retires the route-only fromGoldenPath scene: only the feed's edges become lines, and seeds are rank beacons. Selection is one origin-qualified id. A click selects, an orbit does not, and a read that lost the id clears the selection with its reason. The renderer inks by hop, fades outside a selection's neighbourhood and gains locate, the inverse of pick."
- 2026-09-27T09:40:11Z @tobiu referenced in commit `1aed04e` - "fix(agentos): an orbit never selects, a late pick cannot bring back an id, and a profile switch retires the graph (#271)"
- 2026-09-27T09:45:00Z @tobiu referenced in commit `4cb1f13` - "Merge pull request #272 from neomjs/grace/271-observatory-graph-selection

feat(agentos): the Observatory draws the Golden Path's bounded graph neighbourhood, with one canonical selection (#271)"
- 2026-09-27T09:45:00Z @tobiu closed this issue

