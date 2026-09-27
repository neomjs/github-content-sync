---
id: 210
title: A Golden Path pane renders the computed route as text with its currency
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-25T15:49:03Z'
updatedAt: '2026-09-27T16:36:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/210'
author: neo-fable
commentsCount: 2
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
closedAt: '2026-09-27T16:26:26Z'
---
# A Golden Path pane renders the computed route as text with its currency

## Context
Reopened for the operator's 2026-09-27 correction: the Golden Path pane must show the complete producer-written recommendation, not just the reduced typed route. Grace transferred this follow-up to Emmy. Authority: [existing org D#19151](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18624317).

## Problem and solution
The current pane omits semantic/structural score breakdowns, routing guard and Strategic Interpretation. Render the exact human Golden Path Markdown section as the primary content using the existing native Markdown component and pane-local SCSS. Replace the duplicate reduced-route cards; retain useful typed-route provenance and REM information as explicitly separate secondary information. The graph continues to consume the typed route.

## Contract Ledger
| Surface | Behavior | Fallback / evidence |
|---|---|---|
| `GoldenPathEnvelope.handoff` | Closed block: `markdown, mtimeMs, ageMs, staleAfterMs, stale, reason` from Brain #496 | Older/missing source renders unavailable; envelope test |
| Primary recommendation | Full producer Markdown, including heading, full titles, scores, guard when present, interpretation and capture line | Native Markdown rendering; no reconstructed prose or scores |
| Freshness | Handoff update/stale state is independent of typed-route admission | Label each axis; never call file mtime the recommendation's capture time |
| Layout | Readable wrapping and ordinary scrolling in both themes | Render inspection at narrow and wide pane sizes |

## Acceptance Criteria
- [ ] Full recommendation content survives the envelope and is readable without clipping.
- [ ] Missing and stale handoff states are explicit; a fresh route never makes an old human section look fresh.
- [ ] A missing human section does not corrupt the typed route, graph selection data or REM state.
- [ ] Both themes render the content with the native Markdown component; focused tests and visual evidence cover the changed pane.

## Post-Merge Validation
- [x] Emmy: installed FM shows the actual producer-written section after packaging the merged reader. This delivery receipt remains tracked by the open parent #10.

## Boundaries
No ranking, synthesis, Observatory changes, new navigation surface or new Markdown parser. Reuse the existing pane and Brain #496. The original typed-route-only implementation is already shipped; these criteria describe the reopened completion.

Origin Session ID: f4539f98-814e-43c1-8214-a10206fb0d73


Installed verification: [delivery receipt, 2026-09-27 16:30–16:32 UTC](https://github.com/neomjs/neo-agent-institution/issues/10#issuecomment-5856530380). The real producer section, full titles and scores, interpretation, scrolling and separate freshness states are visible in the installed app.



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
- 2026-09-26T07:21:12Z @neo-fable-clio cross-referenced by #228
- 2026-09-26T07:28:22Z @neo-fable-clio cross-referenced by #230
- 2026-09-26T08:05:14Z @neo-fable-clio cross-referenced by PR #233
### @neo-gpt-emmy - 2026-09-27T14:27:05Z

Picking up the bounded follow-up transferred by Grace in A2A `MESSAGE:96605579-5993-441d-ac8c-505b61d6c867`, under the operator's current Golden Path correction and [D#19151](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18624317).

The pane must show the whole producer-written recommendation: full titles, total and semantic/structural scores, routing guard when present, Strategic Interpretation, and source freshness. The reduced typed route does not supply that content. Reuse the existing pane and native Markdown renderer; preserve the typed route's independent admission and provenance. Companion read completion is existing Brain #496. No Observatory edits or new ticket.

- 2026-09-27T14:27:07Z @neo-gpt-emmy reopened this issue
- 2026-09-27T14:27:09Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-27T14:27:09Z @neo-gpt-emmy unassigned from @neo-opus-grace
### @neo-gpt-emmy - 2026-09-27T14:27:10Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-grace`
**New assignees:** `neo-gpt-emmy`
**Reason:** Grace explicitly transferred the closed issue's complete human recommendation follow-up in MESSAGE:96605579-5993-441d-ac8c-505b61d6c867; original graph implementation stays with Grace.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-09-27T14:56:09Z @neo-gpt-emmy cross-referenced by #496
- 2026-09-27T15:03:01Z @neo-gpt-emmy cross-referenced by PR #586
- 2026-09-27T15:16:40Z @neo-gpt-emmy cross-referenced by #13018
- 2026-09-27T15:24:15Z @neo-gpt-emmy cross-referenced by PR #19313
- 2026-09-27T15:54:30Z @neo-gpt-emmy cross-referenced by PR #304
- 2026-09-27T16:13:05Z @tobiu referenced in commit `3900f91` - "feat(fleet): read the complete Golden Path recommendation (#210)"
- 2026-09-27T16:13:06Z @tobiu referenced in commit `32c59d3` - "fix(fleet): detach the recommendation reader from graph selection (#210)"
- 2026-09-27T16:26:26Z @tobiu referenced in commit `68435c1` - "Merge pull request #304 from neomjs/codex/210-complete-gp-content

feat(fleet): read the complete Golden Path recommendation (#210)"
- 2026-09-27T16:26:27Z @tobiu closed this issue
- 2026-09-27T16:34:52Z @neo-gpt-emmy cross-referenced by #10

