---
id: 546
title: 'fleetGraphScene: deliver the Brain-side bounded scene feed (the AC-1..3 slice of #533)'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-26T11:29:07Z'
updatedAt: '2026-09-26T11:29:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/546'
author: neo-preview
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
# fleetGraphScene: deliver the Brain-side bounded scene feed (the AC-1..3 slice of #533)

## Problem

#533 spans two repositories: the Brain-side feed, and a post-merge receipt on the pane side. That
makes it un-closeable by a Brain PR, because closing it would assert a cross-repo AC this repo cannot
deliver. The shared `pr-body` baseline enforces a real close target, so the Brain-side slice needs its
own leaf to be closed honestly.

This ticket is that leaf: **exactly** the Brain-side AC-1/AC-2/AC-3 delivery. #533 stays open as the
umbrella, carrying AC-4.

## Scope

- `ai/services/fleet/fleetGraphSceneSource.mjs` — the source, the pure projector, the read reducer.
- `ai/services/fleet/wireFleetGraphSceneSource.mjs` — the wiring, unwired-slot-is-honest posture.
- `ai/services/fleet/FleetControlBridge.mjs` — the `graphSceneSource` slot and `fleetGraphScene` method.
- `ai/services/fleet/fleetServerPolicy.mjs` — the `awaiting-s3` / `read-observe` rows.
- `ai/services/fleet/devFleetServer.mjs` — the server-entry wiring.
- `src/fleet/contract/wire.mjs` — `FLEET_WIRE_METHODS`.
- Spec + the two parity ledgers + the brain-unit arm list.

## AC

- [ ] AC-1 `fleetGraphScene({seeds?, depth?, maxNodes?, maxEdges?})` answers the envelope through the
      fleet wire from a `devFleetServer`, with the computed route's items as the default seeds.
- [ ] AC-2 RLS holds: an unauthorized neighbour and its edges are absent, and `completeness` reads
      `truncated` only for budget cuts — never for a scope cut.
- [ ] AC-3 Ids are origin-qualified and stable across two reads of the same route (spec).

## Deliberately out of scope

- **A relation label on edges.** The neighbour projection drops every edge property except `weight`,
  so a `type` is absent data, not weak data. Defaulting it would render every real edge in the
  cockpit as the same invented relation. A feed needing relation semantics reads raw edge rows.
- **A continuation token.** Budgets are mandatory; the envelope's `snapshotId` is the identity a
  future resumable read would bind. A token without that binding makes two reads of one snapshot look
  like two snapshots.
- **AC-4**, which stays on #533 with the pane side.

Refs #533

## Timeline

- 2026-09-26T11:29:08Z @neo-preview assigned to @neo-preview
- 2026-09-26T11:29:09Z @neo-preview added the `enhancement` label
- 2026-09-26T11:29:09Z @neo-preview added the `ai` label
- 2026-09-26T11:29:18Z @neo-preview referenced in commit `fd10300` - "fix(fleet): hand the walked links to the projector — the feed emitted a graph with no edges (#546)

`readGraphScene` collected nodes from the adjacency it walked and never passed the links
to `projectNeighbourhood`, so every read returned a scene with `edges: []`. A 3D graph
with no links.

The reason it survived 31 local arms is the reason it is worth writing down: **every
edge assertion called `projectNeighbourhood` DIRECTLY**, and two arms asserted
`scene.edges` was empty — which an edge-less feed satisfies for the wrong reason. That is
the same shape as a budget arm kept green by `toBeLessThanOrEqual(4)` against an empty
scene: an assertion that cannot distinguish "the mechanism removed it" from "the
mechanism never produced it".

Two changes:

- The walk now records each hop as a link BEFORE the already-seen check, so a hop back to
  a node already in the scene survives. Keying the record off `seen` would silently delete
  every cycle and every cross-link in the graph.
- A new spec arm reaches edges through the SOURCE and pins a cycle surviving, at depth 2
  (at depth 1 the neighbours are discovered but never resolved, so they are legitimately
  absent and their links correctly drop as dangling — which is a distinction worth having
  in a test rather than in a comment).

Also drops a legacy `(#314)` from a comment in `dispatchFleetRequest.spec.mjs`. That
violation predates this work, but the archaeology check is file-scoped rather than
diff-scoped, so touching the file inherited it; the rule it enforces says provenance
belongs in the commit, not in source."
- 2026-09-26T11:29:53Z @neo-preview cross-referenced by PR #545
- 2026-09-26T11:37:35Z @neo-preview referenced in commit `034c96c` - "test(fleet): the cycle arm contradicted the sort the projector documents (#546)

CI had it right and the arm wrong: all three edges were present, and the two links out
of pr-101 were ordered issue-9 before issue-7. The projector sorts by `from`, then `to`,
ascending, so issue-7 precedes issue-9. I hand-wrote the expectation and it disagreed
with the rule two functions away in the same file.

The local probe passed because it asserted the cycle's EXISTENCE and the first edge's
`from`, never the full order — weaker, not wrong. It now pins all three in sequence, so
the next ordering slip is caught outside CI rather than by it.

Worth naming as a pair with the defect it followed: the previous commit fixed a real
missing-edges bug that 31 arms missed, and this one is an ordering slip in the arm that
caught it. Both are the same underlying habit — writing what the output *should* look
like from memory instead of deriving it from the rule the code states."
- 2026-09-26T11:58:10Z @neo-preview referenced in commit `d257704` - "fix(fleet): three seam contracts were the source's expectations, not the wire's (#546)

Cross-family review measured the three graph operations live and found the seam layer
contradicting all of them. Every finding is confirmed here, and two of the three were
errors I had introduced and then defended in the module doc.

**The route read.** The operation answers an ENVELOPE — `{status, reason, details, route,
admission}` with `status` one of `available | missing | …` and freshness nested at
`route.route.status`; the item id is `id`, not `ref`. This compared the top level to
'fresh', which can never hold, so every wired read answered `route-not-fresh`. The
sibling route source already reduces this exact shape. A missing route is now `degraded`
with the operation's own reason rather than `unavailable`: there is no graph to be
unavailable from when only the route is absent, so the severity was wrong too.

**The ids.** Live: `get_node('issue-9853')` answers the node, `get_node('neomjs/neo#issue-9853')`
answers `{result: null}`. The seam is keyed by the graph's BARE id; qualification belongs on the
EMITTED scene, the only place the origin is known. The previous doc claimed the opposite and
made the stub enforce it — which is the failure mode worth recording: a stub derived from
the thing under test cannot disagree with it, so hardening the stub hardens the error.
The stub is now transcribed from a recorded live answer.

**The edges.** The neighbour operation names the direction on the edge (`source` / `target`)
and the relation as `relationship`. Deriving `{from: the node we asked about, to: the
neighbour}` reverses every INBOUND edge, and because the operation answers an edge from both
endpoints, it also emits each edge twice — spending the edge budget twice on unique links.
Direction and relation are now read off the edge, and links are deduplicated on the oriented
pair so a genuine parallel edge with a different relation stays two edges.

**Depth was off by one.** The seeds are hop zero, so the walk runs `depth + 1` levels.
Bounding at `level < depth` made `depth: 1` — the default — return the seeds alone while
reporting `depth: 1`, with the scene still claiming `complete` and the first ring missing.

**Two further defects found while fixing these, both mine:**

- `await getNeighbors(id)?.neighbors` applies the optional chaining to the PROMISE, not to
  the resolved answer, so the expression is `await (promise.neighbors)` — always `undefined`.
  It failed silently: no throw, no error path, just a scene with no edges. The await and the
  member access are now separate statements, with the reason recorded.
- A seam that RAISES is not a seam that WITHHOLDS. `get_node` answers `null` for a row the
  reader may not see and raises when the store cannot answer at all. Both were collapsed into
  a scope cut, which reports an infrastructure outage as a permission — and `complete` on top
  of it launders the outage into a fact about the graph. A refusal is now `degraded` with the
  operation's reason and hands over the partial scene it did resolve.

The spec's fixtures are transcribed from recorded live answers, and four arms now cover what
the previous shape made invisible: the route envelope, a refused seam, the depth off-by-one,
and inbound direction with single-counted edges."
- 2026-09-26T11:58:43Z @neo-preview referenced in commit `a6c8b1f` - "perf(fleet): resolve the final ring without expanding its adjacency (#546)

RA-2 asked for the boundary ring to be resolved but not expanded, and the first pass
resolved it and then walked its adjacency anyway — a real call against a real store for
a ring the scene cannot contain. The depth boundary now stops the expansion, so a
depth-d read issues d adjacency calls rather than one per resolved node."
- 2026-09-26T12:02:20Z @neo-preview referenced in commit `e99671c` - "test(fleet): three arms of the scene spec could not run at all (#546)

CI: 5 failed, 1977 passed — all five in this spec, and only three are logic.

**Two arms asserted the superseded contract.** One asserted an emitted edge carries no
relation, which was the premise RA-3 retired: the live answer supplies it as
`relationship`, so it is present data. It now asserts the narrower invariant that actually
matters — a supplied relation is passed through, an absent one is ABSENT rather than
defaulted. The other fed the PURE projector the wire's `{source, target}` shape; the
projector takes internal `{from, to}` links, and translating between them is the source's
job. It was reaching `undefined.from` and reporting a TypeError instead of a diff.

**Three arms never ran, for a reason that has nothing to do with this module.**
`FleetControlBridge` and the policy ledgers reach `neo.mjs`, whose modules call
`Neo.gatekeep` at evaluation time, and they were static imports — so they evaluated
against whatever ambient global the worker inherited. The sibling spec makes the same
static imports and passes, because a different spec ran first in ITS worker and left the
global behind. So whether these three arms executed depended on which worker they landed
in: an arm that runs or does not depending on file order is not an arm.

They now load dynamically, after the framework entrypoint establishes the global it
assigns. This is also the honest answer to the evidence correction earlier in this PR: the
tier does run — CI collects it — so "the ai/** tier cannot run locally" was the seat
condition I described, not a property of the harness, and the arms that need the graph
ledgers have always been runnable in CI."
- 2026-09-26T12:02:53Z @neo-preview referenced in commit `fbc0678` - "test(fleet): remove the superseded edge arm and fix the projector's input local (#546)

Follow-up to the previous commit, which pushed two defects of its own:

- Replacing the arm that asserted an edge carries no relation INSERTED the new arm
  beside the old one instead of replacing it, so the retired assertion was still in the
  file and would still have failed.
- The projector's input local was renamed to `links` at the use sites but not at its
  declaration, leaving a guaranteed ReferenceError where the arm previously reported a
  TypeError. Same shape as the seam defects this PR is about: the edit was applied
  against my expectation of the file rather than against the file.

Verified before committing this time — balanced delimiters, every helper that is called is
defined, 19 arms, and no bare `edges` local left dangling. The previous commit was pushed
on a sanity check whose output I read as passing when it had already said `false` twice."
- 2026-09-26T12:13:56Z @neo-preview referenced in commit `baa8ae0` - "test(fleet): remove a duplicated test title that blocked collection of the whole spec (#546)

Playwright refuses to collect a file with two arms of the same title, so the duplicate
took down all 18 other arms in this spec — the run reported one error rather than one
failure. The duplicate came from the same index-based block surgery as the two defects in
the previous commit, which inserted the new arms beside the old ones instead of
replacing them.

The verification I ran before that commit checked that delimiters balanced, that every
helper called was defined, and counted the arms. All three passed, and none of them can
see a duplicate: I checked the file's STRUCTURE and not its UNIQUENESS. Uniqueness is now
asserted directly, alongside the structural checks, since a title collision is a
collection-time failure that no per-arm assertion would ever report."
- 2026-09-26T12:31:08Z @neo-preview referenced in commit `707756c` - "fix(fleet): resolve both edge endpoints off the edge, never off the node asked about (#546)

Round 2, measured with the plane's own `get_neighbors` answer: an inbound edge names a
`source` that is not the node being asked about, and usually is not in the read either.
The walk resolved the source with a `?? qualifyNodeId(askedAbout)` fallback, so `from`
collapsed onto the expanded node and every inbound edge came back as a self-loop —
`issue-19235 -> issue-19235` — carrying a real relation, which is the shape least likely
to be caught by eye and impossible to catch in a stub whose edges are all outbound.

Both endpoints now come off the edge. The two cases are asserted separately because they
differ and both are correct: an in-scene inbound edge draws in the graph's own direction,
and an out-of-scene one is dropped as a dangling cross-link rather than drawn backwards.
A fallback that substitutes the wrong endpoint is worse than no fallback — it does not
degrade the scene, it fabricates an edge, and it fabricates one that looks legitimate.

Local receipt: 36 arms, all passing."
- 2026-09-26T13:00:25Z @neo-preview referenced in commit `1b5ae64` - "fix(fleet): register and enqueue the edge's other endpoint before qualifying it (#546)

The self-loop fix addressed the SYMPTOM of the inbound defect, not its whole cause, and
my Round 2 response claimed the Required Actions were discharged when they were not.
Grace named both halves; I had fixed one.

Two ordering defects, and they compounded:

- The guard tested `byId.has(neighbour.target)` while the write keyed `byId.set(neighbour.id)`.
  Guard and key disagreed, so for an INBOUND edge — where `target` is the node already
  expanded — the guard was false and the outside endpoint was never registered.
- `seen.has(to)` decided the enqueue, and for an inbound edge `to` IS the node just
  expanded, so the walk enqueued nothing and `?? neighbour.target` papered over it by
  offering the same node again. The far endpoint stayed unknown to the walk at any depth,
  so its edge was dropped as dangling even when a deeper read should have carried it.

`neighbour.id` is always the edge's other endpoint, so it is now registered and enqueued
FIRST, and `source` / `target` are qualified directly afterwards. Three arms cover it: no
self-loop at depth 1, direction preserved when both endpoints are in the scene, and at
depth 2 the far endpoint is reached and its edge becomes drawable.

Local receipt: 38 arms, all passing."

