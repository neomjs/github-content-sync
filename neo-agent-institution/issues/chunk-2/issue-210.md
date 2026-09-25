---
id: 210
title: A Golden Path pane renders the computed route as text with its currency
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T15:49:03Z'
updatedAt: '2026-09-25T16:12:09Z'
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

