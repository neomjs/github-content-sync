---
id: 210
title: A Golden Path pane renders the computed route as text with its currency
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T15:49:03Z'
updatedAt: '2026-09-25T18:57:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/210'
author: neo-fable
commentsCount: 0
parentIssue: 9
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T18:57:48Z'
---
# A Golden Path pane renders the computed route as text with its currency

## Context

The operator's refocus of 2026-09-25 names "GP and graph inside FM" as the high-ROI work, and the lead's lane 3 asks for a Golden Path text representation in the Fleet Manager. The Brain side (a `fleetGoldenPath` fleet-wire read returning the computed route with its freshness, the corpus-projection admission and the REM counts) is filed in neo-agent-brain; this ticket is the cockpit half.

## The Problem

The Golden Path is the picture the institution steers by, and the cockpit does not show it. Today the underlying read is withheld (the corpus projection is not current; REM has 990 undigested turns and no recent cycle), which is exactly what the pane must say: a route presented without its currency would mislead, and a pane that hides a withheld state would hide the day's real problem.

## The Architectural Reality

- `apps/agentos/view/fleet/catchup/Container.mjs`: the south-strip reading-surface precedent — renders source-owned `notAuthority` envelopes without synthesizing, ranking, merging or caching; honest states (unavailable / degraded / empty, generated timestamp, coverage, citations) are first-class; reads go through intent events the owning cockpit relays to the authenticated bridge (`cockpit/Controller.mjs` ~:239, `cockpit/Container.mjs` ~:191 registers the pane).
- `apps/agentos/fleet/installFleetBridge.mjs`: the app↔fleet transport over the installed public contract; the new method arrives with the Brain package bump.
- The design system (#13) and the visual-regression baseline harness (#11) govern the pane's look and its goldens.

## The Fix

1. `apps/agentos/view/fleet/goldenpath/Container.mjs`: a south-strip tab "Golden Path" rendering the `fleetGoldenPath` envelope as text: a currency line first (current at `capturedAt` / last known good, captured `capturedAt`, withheld since `reasonCode` / unavailable with the capability reason; REM undigested / digested / recent cycles beside it), then the route items in the producer's order (rank, id, title, score, the reasons as written), then provenance (`producer`, `runId`, `algorithmVersion`, `expiresAt`). No synthesis, no re-ranking, no caching beyond the pane's own store.
2. Registration beside the catch-up pane; the cockpit controller relays the read intent and a refresh; the liveness controller's refresh cadence may include it (same as catch-up).
3. Goldens for the four states (current / last-known-good / withheld / unavailable) in both themes through the #11 harness; unit spec on the pane's rendering of a fixture envelope per state.

## Acceptance Criteria

- [ ] AC-1 With the Brain read returning a fixture envelope, the pane renders each of the four states with the currency line first and the items in producer order; unit spec per state.
- [ ] AC-2 Against the live plane the pane shows the real state of the day (today: withheld with `freshness-sla-breached` and the last route's `capturedAt`), never a route presented as current.
- [ ] AC-3 Goldens for the four states in both themes pass the #11 harness; the pane uses the token system (#13), no pixel literals.
- [ ] AC-4 The pane synthesizes nothing (reviewed against the diff).

## Out of Scope

The Brain read itself; a graph rendering of concepts or edges (Institution #8's COP); ranking changes to the Golden Path (Brain #122's leaves).

## Related

neomjs/neo-agent-institution#9 (parent: the keeper views), #10 (the design-led surface), #13, #11, the Brain read ticket (linked from the broadcast), neomjs/neo-agent-brain#122.

unowned-rationale: the Brain read lands first; the pane is the FM design seat's kind of work — @neo-fable-clio or @neo-preview claim it with a [lane-claim], and I take it myself once the read is on a plane pin if nobody has.

Live latest-open sweep: checked the latest 20 open issues of this repository and of neo-agent-brain at 2026-09-25T15:47:01Z; no equivalent. A2A in-flight sweep (15:44Z) and my [lane-intent] broadcast at 15:46Z: no competing claim. Memory Core sweep: no prior record. Own-assignment sweep: none of mine covers it.

Retrieval Hint: `query_raw_memories("Golden Path pane cockpit south strip currency line fleetGoldenPath")`

Origin Session ID: 4c0a5550-17ba-4752-9852-846afa537c86

## Timeline

- 2026-09-25T15:49:05Z @neo-fable added the `enhancement` label
- 2026-09-25T15:49:05Z @neo-fable added the `ai` label
- 2026-09-25T15:49:05Z @neo-fable added the `design` label
- 2026-09-25T15:49:23Z @neo-fable added parent issue #9
- 2026-09-25T15:57:45Z @neo-fable-clio cross-referenced by #211
- 2026-09-25T16:12:09Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T16:43:53Z @neo-opus-vega cross-referenced by #500
- 2026-09-25T16:56:54Z @neo-fable-clio cross-referenced by #213
- 2026-09-25T17:01:29Z @neo-preview cross-referenced by PR #502
- 2026-09-25T17:18:53Z @neo-opus-grace cross-referenced by PR #215
- 2026-09-25T17:31:24Z @neo-opus-vega cross-referenced by PR #499
- 2026-09-25T17:48:05Z @neo-opus-grace referenced in commit `0466ecd` - "feat(cockpit): a Golden Path pane renders the computed route as text with its currency (#210)

The south strip gains a Golden Path tab. It renders the fleetGoldenPath envelope in a fixed order:
- first, a currency line in the producer's words;
- then the route's items, in the producer's order, from a pane-local Store;
- last, the producer's provenance.

The pane synthesizes, ranks and caches nothing.

A route reads as current only when three things hold: the admission admits it, the route is fresh,
and it has not expired. Otherwise it is withheld and shown as the last known good route, with the
producer's reason. A degraded source (route file missing or invalid) and an unwired or failed read
say so instead of showing an empty route.

The envelope is cockpit provider truth, because more than one pane reads it (the graph pane of #213
binds the same leaf). The read lands it in the goldenPathEnvelope leaf through GoldenPathEnvelope,
which also holds the currency both panes derive. The landing is closed:
- every declared key is written on every read, so a block the wire sends as null lands as its blank;
- the reason is that setData drills objects into leaf paths, and an ancestor rebuild stops at a null
  block, while an omitted key keeps its old value;
- a real-provider arm pins this, with the raw wire envelopes as its control.

A bound envelope can land while the pane is still constructing, so the pane applies an envelope once
its Store exists rather than once construction completes.

The cockpit Controller (998 lines) and Container (999) sat at the 1,000-line bar, so two
responsibilities move out whole:
- the catch-up and Golden Path owners become a ReadingSurfacesController layer between the liveness
  layer and the intent layer;
- the perspective share beat's logic moves into CockpitPerspectives, with thin wrappers left
  behind.

The Neural Link spec lands the pane's four states through the cockpit's own write and captures them
in both skins. The three cockpit baselines that carry the stream tab strip are re-rendered: the new
tab sits under the diff threshold, so they passed without it."
- 2026-09-25T17:58:01Z @neo-fable-clio cross-referenced by PR #216
- 2026-09-25T18:00:22Z @tobiu referenced in commit `e2d756d` - "fix(cockpit): the visual stamp covers the Golden Path inputs, and the shape comment names no ticket (#210)

The golden stamp digests the committed apps/agentos inputs, which the Golden Path util and pane changed; the visual suite and the pane's Neural Link goldens pass unchanged at this head. The util's shape comment described its source by ticket; it now names the Brain's wire, which the source-comment archaeology check requires."
- 2026-09-25T18:57:48Z @tobiu closed this issue

