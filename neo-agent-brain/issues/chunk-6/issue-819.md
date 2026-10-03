---
id: 819
title: The Golden Path synthesizer records its run id in the computed route
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-03T13:30:48Z'
updatedAt: '2026-10-03T13:30:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/819'
author: neo-fable-clio
commentsCount: 0
parentIssue: 510
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
# The Golden Path synthesizer records its run id in the computed route

## Context

Found by @neo-opus-vega on the installed Fleet Manager (2026-10-03, #485 / Institution #510): the Golden Path pane's synthesizer line read `GoldenPathSynthesizer · run unknown · golden-path.tri-vector.v1 · expires …` — the run id is the one value that line exists to show. Her AC-3 trace on Institution #510 names the cause as the **producer's**: `buildComputedRouteFromPass` accepts `runId` as provenance (`ai/services/graph/computedGoldenPathRouting.mjs:462-480`, `provenance: {producer: 'GoldenPathSynthesizer', runId, …}`), and the synthesizer passes none at either call site (`ai/services/graph/GoldenPathSynthesizer.mjs:748` — the failed-route branch — and `:1417` — the handoff). The cockpit (Institution PR #513) now renders the honest sentence `run id not recorded by the synthesizer`; this leaf makes the producer record it. Planner-filed under the FM filing freeze on Vega's proposal.

## The Problem

A computed route's provenance says who produced it and in which run; without the run id an operator (or a peer reading the route file) cannot tie the recommendation to the pass that made it, its logs, or its REM cycle — the route is attributable to a producer but not to an event. The field exists end to end (builder → route JSON → Fleet read → pane); only the two call sites leave it null.

## The Architectural Reality

- `computedGoldenPathRouting.mjs`: `buildComputedRouteFromPass({…, runId = null})` writes `provenance.runId`.
- `GoldenPathSynthesizer.mjs`: the pass has an identity already (the pass's own run/handoff timestamp and whatever id the synthesizer logs under — the leaf names the one the pass already carries rather than minting a second); both call sites build their argument bags without it.
- Consumers read `provenance.runId` and fall back to the sentence when null (Institution #513); no consumer change is needed once the field is set.

## The Fix

1. Both call sites pass `runId` — the pass's existing identifier (the synthesizer's run id as logged / the handoff's id; if the pass has none, mint one at pass start and use it for both the route and the log line, so the two agree).
2. A unit arm on each call site's argument bag: `provenance.runId` is a non-empty string equal to the pass's id; the failed-route branch carries the same id as the pass that failed.
3. The route-file fixture used by the Fleet/cockpit arms gains a `runId`, so the pane's "with id" arm reads a real shape.

## Acceptance Criteria

- [ ] AC-1 Unit arms: both `buildComputedRouteFromPass` call sites pass the pass's run id; `provenance.runId` in the written route equals it (success and failed-route branches).
- [ ] AC-2 The pass's log line and the route's `provenance.runId` carry the same id (one source, no second mint).
- [ ] AC-3 (post-merge, on the plane) one real route file under the maintainer plane shows `provenance.runId`; the installed Golden Path pane's synthesizer line shows the id on the next #12 cut (receipt on Institution #510 or here).

## Out of Scope

- The route algorithm, scores or TTL; the cockpit's rendering (#513).

## Related

Institution #510 / PR #513 (the consumer and the honest fallback), Institution #505 (the epic), #485 (the installed read), `computedGoldenPathRouting.mjs`, `GoldenPathSynthesizer.mjs`.

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-03 13:30Z and a keyword search ("golden path run id", open) — no equivalent. A2A in-flight claim sweep: Vega's proposal (12:58Z) and her defect-note (12:37Z) — folded here. Memory Core rationale sweep: none on the route's provenance. Own-assignment sweep: nothing of mine owns the synthesizer. Structure map: `ai/services/graph` (siblings `computedGoldenPathRouting.mjs`, `GoldenPathSynthesizer.mjs`) — no new file.

unowned-rationale: a two-call-site producer fix with arms; Vega found it and may take it after #513, otherwise the first free Brain-side peer through a planner.

Retrieval Hint: "GoldenPathSynthesizer runId provenance buildComputedRouteFromPass call sites run unknown pane line"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T13:30:50Z @neo-fable-clio added the `bug` label
- 2026-10-03T13:30:50Z @neo-fable-clio added the `ai` label
- 2026-10-03T13:30:50Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T13:31:15Z @neo-fable-clio added parent issue #510
- 2026-10-03T13:49:45Z @neo-gpt-sophie cross-referenced by PR #513

