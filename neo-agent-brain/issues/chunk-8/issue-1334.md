---
id: 1334
title: 'Add Focus (v1) handoff section, Fleet focus axis, and host-edge reader seam'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-28T04:34:13Z'
updatedAt: '2026-10-28T04:52:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1334'
author: neo-gemini-maryam
commentsCount: 1
parentIssue: 123
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# Add Focus (v1) handoff section, Fleet focus axis, and host-edge reader seam

## Context

Split from `neomjs/neo-agent-brain#957` (`A focus section in the Sandman handoff: v1 rows, views, reach, the word`, parent epic `neomjs/neo-agent-brain#123`) per Round 2 review disposition `PRR_kwDOUBzDFM8AAAABUmx6ng` [QUARANTINED_URL: github.com] on `neomjs/neo-agent-brain#1332`.

Because `check-pr-body.mjs` (`neo-agent-skills-pr-body --close-target-only`) requires every PR to carry one standalone `Resolves #N` close target while `neomjs/neo-agent-brain#957` must remain open until `neomjs/neo-agent-brain#1333` delivers the cross-plane host-edge dated projection and live seat verification (`AC-5` host-plane projection half and `AC-6`), this leaf owns the exact slice delivered by `neomjs/neo-agent-brain#1332`:
- the `## Focus (v1)` (`focus.v1`, `<= 2,000` UTF-8 bytes) builder, view receipt evaluator, and `publicFlag` business-row redactor (`ai/services/graph/focusV1Section.mjs`), wired into `GoldenPathSynthesizer#synthesizeGoldenPath`;
- the independent `focus` axis in `ai/services/fleet/fleetGoldenPathSource.mjs` (`readGoldenPath` envelope `focus` + `sources.focus`);
- the cloud-plane refusing reader (`makeRefusingFocusV1Reader(CLOUD_PLANE_HOST_EDGE_ABSENT_REASON)` when `repoEnrichmentEnabled === false`) and plane-neutral unconfigured fallback (`DEFAULT_HOST_EDGE_ABSENT_REASON`) in `ai/daemons/orchestrator/scheduling/pipeline.mjs` without spawning `gh` inside the container plane;
- the host-edge block reader and deterministic `gh` parser seam (`createFocusV1HostEdgeReader`, `createDefaultFocusV1HostEdgeReaders`, `parseRoadmapV1Rows`, `parseViewsReceiptComments`, `parseReachTrafficPayload`, `parseCandidatePrWordPayload`) consumed by `neomjs/neo-agent-brain#1333`.

- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1333` down to `#875`) at `2026-10-28T04:33:14Z`; no duplicate of this delivered-slice seam leaf found (`#957` is the parent focus-section ticket; `#1333` is the host-edge dated projection delivery leaf).
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', limit: 30})` at `2026-10-28T04:33:17Z`; no competing `[lane-claim]` or `[lane-intent]`.
- **MC rationale sweep:** `query_raw_memories({query: "Focus v1 handoff section Fleet focus axis cloud plane refusing reader host-edge parser seam", nResults: 5})` at `2026-10-28T04:33:19Z` (5 results, no conflicting decision).
- **Own-assignment sweep:** `#957` assigned to `@neo-gemini-maryam`.
- **Structure Map Gate (`§1c`):** ran `node ./ai/scripts/diagnostics/structureMap.mjs --files --loc`; owning modules are `ai/services/graph/focusV1Section.mjs`, `ai/services/graph/GoldenPathSynthesizer.mjs`, `ai/services/fleet/fleetGoldenPathSource.mjs`, and `ai/daemons/orchestrator/scheduling/pipeline.mjs` (sibling fast-path under `ai/services/graph/`).

## The Problem

The Sandman handoff previously lacked a bounded `## Focus (v1)` section summarizing the four `v1` focus blocks (`v1 rows`, `views`, `reach`, `the word`), and `fleetGoldenPathSource` (`readGoldenPath`) extracted only `## Computed Golden Path (Strategic Recommendation)` and `laneLandscape`, leaving Fleet consumers without an independent `focus` axis. At the same time, container-plane synthesis passes cannot spawn `gh` directly because the `local-agent-os` container plane holds neither `gh` nor GitHub credentials (`neomjs/neo-agent-brain#957` boundary clause).

## The Architectural Reality

- `ai/services/graph/GoldenPathSynthesizer.mjs` synthesizes the Sandman handoff markdown document; extracting the `## Focus (v1)` builder and parsers into `ai/services/graph/focusV1Section.mjs` avoids monolith accretion while keeping static delegates on `GoldenPathSynthesizer`.
- `ai/services/fleet/fleetGoldenPathSource.mjs` parses the handoff markdown and projects independent axes (`computed`, `laneLandscape`, and now `focus`) with per-axis `sources` metadata.
- `ai/daemons/orchestrator/scheduling/pipeline.mjs` (`runGoldenPathTask`) and `ai/services/memory-core/MemoryService.mjs` (`mutateFrontier`) run inside the container plane and must refuse on `repoEnrichmentEnabled === false` (`CLOUD_PLANE_HOST_EDGE_ABSENT_REASON`) or degrade with `DEFAULT_HOST_EDGE_ABSENT_REASON` when no host-edge reader or admitted projection is supplied.

## The Fix

1. Add `ai/services/graph/focusV1Section.mjs` implementing `buildFocusV1Section`, `parseRowStateLine`, `evaluateViewReceipts`, `projectPublicBusinessRows`, `createFocusV1HostEdgeReader`, `createDefaultFocusV1HostEdgeReaders`, `makeRefusingFocusV1Reader`, and the four deterministic `gh` payload parsers (`parseRoadmapV1Rows`, `parseViewsReceiptComments`, `parseReachTrafficPayload`, `parseCandidatePrWordPayload`), wired into `GoldenPathSynthesizer#synthesizeGoldenPath`.
2. Project `## Focus (v1)` onto `focus` + `sources.focus` in `ai/services/fleet/fleetGoldenPathSource.mjs` (`extractFocusV1Section`, `readGoldenPath`).
3. Wire `ai/daemons/orchestrator/scheduling/pipeline.mjs` (`runGoldenPathTask`) to supply `makeRefusingFocusV1Reader(CLOUD_PLANE_HOST_EDGE_ABSENT_REASON)` when `repoEnrichmentEnabled === false` and omit in-plane `gh` execution on container-plane passes.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ai/services/graph/focusV1Section.mjs` & `GoldenPathSynthesizer#synthesizeGoldenPath` | `neomjs/neo-agent-brain#957` (`focus.v1`) | Appends bounded `## Focus (v1)` (`<= 2,000` UTF-8 bytes) with raw steward `Row state:` words, installed-over-source `#505` view receipt evaluation, and fail-closed `publicFlag: true` business-row projection | Renders `unknown` with plane-neutral `DEFAULT_HOST_EDGE_ABSENT_REASON` or `CLOUD_PLANE_HOST_EDGE_ABSENT_REASON` when inputs are absent | JSDoc on `focusV1Section.mjs` and `GoldenPathSynthesizer.mjs` | `test/playwright/unit/ai/services/graph/FocusV1Section.spec.mjs` + `GoldenPathSynthesizer.spec.mjs` |
| `ai/services/fleet/fleetGoldenPathSource.mjs` (`extractFocusV1Section`, `readGoldenPath`) | `neomjs/neo-agent-brain#957` AC-4 | Extracts `## Focus (v1)` onto `focus` + `sources.focus` without coupling to `computed` or `laneLandscape` | Degrades `focus` axis with `handoff-section-not-found` when `## Focus (v1)` is absent | JSDoc on `fleetGoldenPathSource.mjs` | `test/playwright/unit/ai/services/fleet/FleetGoldenPathSource.spec.mjs` + `FleetActivityReadEndToEnd.spec.mjs` |
| `ai/daemons/orchestrator/scheduling/pipeline.mjs` (`runGoldenPathTask`) | `neomjs/neo-agent-brain#957` AC-5 boundary clause | Supplies `makeRefusingFocusV1Reader(CLOUD_PLANE_HOST_EDGE_ABSENT_REASON)` when `repoEnrichmentEnabled === false` and omits in-plane `gh` reader construction on container-plane runs | Plane-neutral `DEFAULT_HOST_EDGE_ABSENT_REASON` until `#1333` wires the admitted dated projection | JSDoc on `pipeline.mjs` | `test/playwright/unit/ai/daemons/orchestrator/scheduling/pipeline.spec.mjs` + `Orchestrator.spec.mjs` |

## Decision Record impact

`aligned-with` ADR 0019 (reads `AiConfig` at the orchestrator pipeline entrypoint without shadow defaults or runtime mutation).

## Acceptance Criteria

- [ ] **AC-1:** `GoldenPathSynthesizer#synthesizeGoldenPath` writes `## Focus (v1)` (`focus.v1`, `<= 2,000` UTF-8 bytes) with all four blocks (`v1 rows`, `views`, `reach`, `the word`), and renders `unknown` with a plane-neutral reason when every input is absent.
- [ ] **AC-2:** `parseRowStateLine` copies steward-owned `Row state:` words raw, truncates at `plan:` tails, and never promotes `ready` to `passed`; `evaluateViewReceipts` enforces installed-over-source governance on `#505` view receipts.
- [ ] **AC-3:** `projectPublicBusinessRows` excludes rows without strict `publicFlag: true`, excludes `BUSINESS_GOAL` nodes, and strips private target fields fail-closed.
- [ ] **AC-4:** `fleetGoldenPathSource` (`readGoldenPath`) projects `## Focus (v1)` onto an independent `focus` axis (`focus` + `sources.focus`) and degrades to `handoff-section-not-found` when absent without affecting `computed` or `laneLandscape`.
- [ ] **AC-5:** Cloud-plane passes (`repoEnrichmentEnabled === false`) supply `makeRefusingFocusV1Reader(CLOUD_PLANE_HOST_EDGE_ABSENT_REASON)`, container-plane passes never spawn `gh` in-plane, and `createFocusV1HostEdgeReader` + `createDefaultFocusV1HostEdgeReaders` provide the tested host-edge reader and parser seam consumed by `#1333`.

## Out of Scope

- Cross-plane host-edge dated projection persistence and live container-plane seat verification (`neomjs/neo-agent-brain#1333` and `neomjs/neo-agent-brain#957`, which stays open until `#1333` lands).

## Avoided Traps

- **Spawning `gh` inside container-plane processes (`pipeline.mjs` / `MemoryService.mjs`):** The `local-agent-os` container holds neither `gh` nor GitHub credentials; host-edge GitHub collection belongs to `#1333` via an admitted dated projection.

## Related

- Parent Epic: `neomjs/neo-agent-brain#123`
- Source ticket (kept open until `#1333` lands): `neomjs/neo-agent-brain#957`
- Sibling host-edge projection leaf: `neomjs/neo-agent-brain#1333`
- PR: `neomjs/neo-agent-brain#1332`

Origin Session ID: 58b07b45-1fe9-40ad-80f8-db0f380332b8
Retrieval Hint: `query_raw_memories("Focus v1 handoff section Fleet focus axis cloud plane refusing reader host-edge parser seam 957 1332 1333")`


## Timeline

- 2026-10-28T04:34:14Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-28T04:34:14Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-28T04:34:14Z @neo-gemini-maryam added the `ai` label
- 2026-10-28T04:34:14Z @neo-gemini-maryam added the `architecture` label
- 2026-10-28T04:34:15Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-28T04:34:17Z @neo-gemini-maryam added parent issue #123
- 2026-10-28T04:36:37Z @neo-gemini-maryam cross-referenced by PR #1332
### @rakesh1566 - 2026-10-28T04:52:26Z

lock

- 2026-10-28T05:18:00Z @neo-gemini-maryam cross-referenced by PR #1337

