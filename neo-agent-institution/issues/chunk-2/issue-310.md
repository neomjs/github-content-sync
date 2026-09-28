---
id: 310
title: The Observatory draws readable wells before any Brain change
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T11:41:47Z'
updatedAt: '2026-09-28T14:12:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/310'
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
blocking:
  - '[ ] 311 The Observatory''s wells follow the roadmap, and the team lens shows who touched what'
---
# The Observatory draws readable wells before any Brain change

## Context

Stage A of D#19317, the Observatory's product definition, which graduated at body `updatedAt 2026-09-28T10:56:25Z` (Signal Ledger below). On the live scene, FM's layout clusters bare topology into 4,638 communities, and the two largest are Ada's and Emmy's inboxes (12,355 and 11,272 nodes, 85–89 % `MESSAGE`). Drawn with FM's own geometry, it is a featureless sphere of tiny communities. On the D#19317 prototype, which builds readable wells from the same data, the operator said: *"a LOT better than what FM shows right now, and closer to clio's PoCs."* This leaf needs **no wire change**: kinds, edge types and edges already travel in `fleetGraphScene`.

## The Problem

- **Mail dominates the geography.** 56 % of the scene is mail. Dropping only the three mail edge types is not enough, because messages stay attached to concepts through `TAGGED_CONCEPT`.
- **Half the scene has no edge** (78,822 nodes on one viewer's scene), and it is drawn as one ball among the communities.
- **Topology communities answer no operator question** (D#19317 Q1). Density wells (W2) at least show what is central, until Stage C brings the Brain's own strategic anchors.

## The Architectural Reality

- `apps/agentos/util/ObservatorySceneLayout.mjs`: unweighted Louvain (`moveLocally`, `aggregate`) over the wire's edges, community centres on a golden sphere (`fromGraphScene`), hash-based positions; deterministic by construction.
- `apps/agentos/util/GraphSceneEnvelope.mjs` `describe(envelope, formatStamp, drawn)`: the head's drawn-versus-read line from `#305` / `#306`.
- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs` `updateLine()` and the route toggle; `apps/agentos/canvas/Observatory.mjs` on the engine's `Neo.canvas.GraphScene` foveated LOD (`#288` / `#291`).

## The Fix

`ObservatorySceneLayout.mjs`:
- **Mail off by default, with a toggle:** drop `MESSAGE` and `BroadcastSentinel` nodes and the `DELIVERED_TO` / `SENT_TO` / `SENT_BY` edges before layout.
- **An outer halo** for nodes with no edge, instead of one community ball.
- **W2 density wells as the default geography:** the top K hubs by degree after the mail filter attract; each node joins the hub it reaches first (breadth-first) and is placed by hop distance. W1 stays selectable as the fallback.
- Placement stays a function of ids and edges only.

`GraphSceneEnvelope.mjs`: the head names what it dropped, with counts ("N messages hidden · M edge-less in the halo"), extending the drawn-versus-read line.

`ObservatoryContainer.mjs`: mail and halo toggles beside the route toggle.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Observatory head line | `GraphSceneEnvelope#describe` | adds the dropped counts (mail, halo) | no count when it is not known (D#19151 OQ6) | module JSDoc | unit arms |
| Observatory geography | `ObservatorySceneLayout` | W2 wells by default, mail off, halo | W1 selectable | module JSDoc | fixture arms, headed check |
| Mail and halo toggles | `ObservatoryContainer` | two toggles, state readable through the Neural Link | mail off, halo on | class JSDoc | NL arm |

## Acceptance Criteria

- [ ] AC-1 The head names what it dropped, with counts, and never shows a total it does not know (D#19151 OQ6), extending the `#306` line.
- [ ] AC-2 Mail is off by default, with a toggle (D#19317 OQ-W2 `[RESOLVED_TO_AC]`); the mail nodes and the mail edges both drop.
- [ ] AC-3 Edge-less nodes sit in an outer halo, toggleable, and the head states its count (OQ-W6).
- [ ] AC-4 W2 wells form on a fixture with hubs; the assignment is identical across two reads of one snapshot; the selection survives a geography or overlay switch.
- [ ] AC-5 Unit arms go red on `dev`; the existing Observatory NL and e2e arms stay green; a 100k fixture lands inside the current first-frame budget.
- [ ] AC-6 (post-merge, installed) `[L4-deferred — operator handoff needed]` Residual-Owner: #312, which carries it after this leaf closes. Headed checks at the operator's viewer from a cold saved-plane launch: first useful paint, selection latency, label and halo readability, resize, graph and route state (D#19317 STEP_BACK point 5). First useful paint depends on the cold `get_graph_scene` read being fixed (D#19317 §7), which is tracked outside this leaf.

## Out of Scope

- W3 strategic wells, heat and the team lens (Stage C, after B1 and B2).
- Any Brain change, and the cold-read defect itself.
- The right-hand panel's Team section (Stage C).

Decision Record: NOT_NEEDED (D#19317: meaning may be computed in the Brain; coordinates stay client-side)
Decision Record impact: none

## Signal Ledger
- `claude`: AUTHOR_SIGNAL by @neo-opus-vega @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGdj`)
- `gpt`: APPROVED by @neo-gpt @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGjY`)

## Unresolved Dissent
(none at the final body anchor)

## Unresolved Liveness
- `unknown` family (@neo-preview, active): no graduation signal. Eos folded a divergence cycle (`DC_kwDODSospM4BHGNj`) and reviews this leaf; not required for quorum.
- `gemini`, `kimi`: `operator_benched`, no signal.

## Discussion Criteria Mapping
- Q1, first cut (density wells) → AC-4; W3 is Stage C
- OQ-W2 mail default → AC-2
- OQ-W6 the unlinked half → AC-3
- STEP_BACK point 5 (headed checks, cold launch) → AC-6
- Eos's preconditions (the head names what it dropped) → AC-1

Related: D#19317 · #10 · D#19151 · `#305` / `#306` · `#288` / `#291`
Reviewer: @neo-preview (handed Stage A over, 2026-09-28)
Live latest-open sweep: latest 20 open Institution issues and latest 8 open Brain issues read at 11:39:59Z, plus an org-wide open-issue search for wells, halo, mail-off, gravityWell and strategicWeight; no equivalent found.
MC sweep: "Observatory featureless sphere inbox wells mail dominates graph", 5 results, no prior decision found.
Own-assignment sweep: 1 open (#308, the System definition), not overlapping.
Origin Session ID: 96f97500-4dcb-461e-bef0-af4e6dc5e24a
Retrieval Hint: "Observatory Stage A mail off halo density wells head names dropped"


## Timeline

- 2026-09-28T11:41:47Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T11:41:49Z @neo-opus-vega added the `enhancement` label
- 2026-09-28T11:41:49Z @neo-opus-vega added the `agent-os` label
- 2026-09-28T11:41:50Z @neo-opus-vega added the `ai` label
- 2026-09-28T11:41:50Z @neo-opus-vega added the `design` label
- 2026-09-28T11:42:16Z @neo-opus-vega cross-referenced by #603
- 2026-09-28T11:42:59Z @neo-opus-vega cross-referenced by #311
- 2026-09-28T11:43:17Z @neo-opus-vega marked this issue as blocking #311
- 2026-09-28T11:52:30Z @neo-opus-vega added parent issue #312
- 2026-09-28T12:07:39Z @neo-gpt-emmy cross-referenced by #312
- 2026-09-28T12:32:04Z @neo-opus-vega cross-referenced by PR #313
- 2026-09-28T13:08:50Z @neo-opus-vega referenced in commit `873141d` - "fix(observatory): tunables are class configs, and the toggles leave the read's line whole (#310)

ObservatorySceneLayout becomes a singleton whose geometry, search bounds,
mail kinds and types, wells and halo are static configs, every step a
method; canvas/Observatory's node sizes, community tones and LOD threshold
join its configs, and hsl/hueOf become methods. Neo.overwrites and a
subclass now reach every value; nothing sits above either class.

The re-captured goldens showed the Mail and Halo toggles clipping the
read's line in the withheld state ("freshness-sla-brea"): the toggles now
end the selection strip, so the head's line keeps its whole width and only
the gesture hint yields, as before. The ghost skin colours a button's label
itself, so the toggles' pressed state never showed; they re-bind the
engine's ghost variables, and a computed-ink assertion witnesses it (red on
the old rules).

Visual goldens re-captured on Darwin, stamp re-written."
- 2026-09-28T14:13:24Z @neo-opus-vega referenced in commit `b1b2bb2` - "fix(observatory): no view filter hides a route seed, and the head counts mail as mail (#310)

With Halo off, a route seed in no well left the scene, and the renderer
joined its neighbours with a path that skipped its rank. Seeds are now known
before either filter runs. A seed in no well keeps its place on the shell,
and a mail seed stays with Mail off, so the route keeps every rank.

The head counted broadcast sentinels as "messages". It now says "mail
nodes", the set the Mail toggle removes.

The controls are red on 873141d's layout and envelope code: the middle
isolated seed, the ordinary halo node that still leaves, the mail seed, and
a sentinel-only head line. Visual stamp re-written; no golden changed."

