---
id: 120
title: Claim-scoped belief revision on the supersede primitive (consumer 3)
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-07-02T19:43:26Z'
updatedAt: '2026-11-02T23:32:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/120'
author: neo-opus-vega
commentsCount: 1
parentIssue: 122
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
# Claim-scoped belief revision on the supersede primitive (consumer 3)

> **Status (2026-11-02, `#1438` backlog sweep — verified at `origin/dev` `613df5d7`):**
> - **Sequencing gates satisfied on `origin/dev`:** Both prerequisites are **CLOSED** and live:
>   1. `neomjs/neo#14474` (closed `2026-07-02`): `ai/services/graph/conceptNeighborhoodProbe.mjs` (read-only concept-neighborhood reachability with four-axis provenance/tier output).
>   2. `neomjs/neo#14502` (closed `2026-07-03`): `ai/scripts/maintenance/auditConceptSpineAliases.mjs` (alias-cluster detection + canonical-id merge; see also `#14540` / `#15125`).
>   *(Duplicate re-file `neomjs/neo#14664` was closed `2026-07-04` in favor of this canonical leaf.)*
> - **Corrected `origin/dev` module & ticket anchors:**
>   1. **Supersede & four-axis fidelity vehicle:** `ai/services/memory-core/CoalescingEngineService.mjs` is the **wake-event coalescing engine** (`ADR 0002 §6.4`), **not** the memory/summary supersession primitive. On `origin/dev`, the four-axis contract (`fidelity: ['sourceTier', 'degraded', 'usedTier']`, `lifecycle: ['lifecycle', 'verifiedAt', 'state', 'supersededBy']`) is defined in `ai/services/graph/conceptNeighborhoodProbe.mjs:32-34`, `sourceTier`/`usedTier` and `superseded` receipt tracking live in `ai/services/memory-core/SessionService.mjs:873-957` and `ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs:53,395-520`, and `neomjs/neo#14418` was transferred to `neomjs/neo-agent-brain#125` (`AC-3 — weak-digest provenance + supersede`).
>   2. **REM / Dream consolidation wiring:** `ai/daemons/orchestrator/services/DreamService.mjs` was extracted in `#215` (`PR #235`) to `src/evolution/RemDigestion.mjs` (`src/evolution/createRemDigestion.mjs`).

## Context

Consumer-3 leaf of epic neomjs/neo-agent-brain#122 — institution-native belief revision, claim-scoped. Filed in the operator-directed full-fledging pass; live latest-open sweep 2026-07-02T19:37:34Z: no equivalent ticket.

The scoping is settled authority (#14422 OQ3, RESOLVED_TO_SCOPING_RULE): belief revision defaults **claim-scoped — the claim class decides the verification surface**; concept-scoped conflict detection remains available as a *query pattern* over claim-anchored data (burden on its advocates); NOT absorbed into `#12679` (bird's-eye navigation may consume outcomes; it is not the granularity SSOT). Implementation vehicle: **extend the four-axis lifecycle/fidelity & summary-supersede substrate, never reinvent** (`ai/services/graph/conceptNeighborhoodProbe.mjs:32-34` `EDGE_CONTRACT_AXES` `fidelity: ['sourceTier', 'degraded', 'usedTier']` + `lifecycle: ['lifecycle', 'verifiedAt', 'state', 'supersededBy']`, `ai/services/memory-core/SessionService.mjs:873-957`, and `ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs:53,395-520`, aligned with `neomjs/neo-agent-brain#125` AC-3; note that `CoalescingEngineService.mjs` is the wake-digest coalescer, not memory supersession).

## The Problem

Two memories disagreeing is invisible until an agent trips over the older one. The swarm's empirical failure classes (the graduation's seven specimen events across five classes — frame-import, parallel-wake self-contradiction, token-collision re-filing, recency-surface undersell, single-snapshot temporal overclaim) are all *governance-caught today* — by peers reading carefully, not by substrate. A stale claim ("v13.1 owner benched", "mailbox healthy", "the flat root must stay empty") keeps surfacing at full confidence after reality moved. Supersession exists mechanically (#14418) but nothing SURFACES candidate conflicts to drive it.

## The Architectural Reality

- Vehicle: `ai/services/graph/conceptNeighborhoodProbe.mjs:32-34` (`EDGE_CONTRACT_AXES` `fidelity` + `lifecycle.supersededBy`) and `ai/services/memory-core/SessionService.mjs:873-957` / `ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs:53,395-520` (`sourceTier`, `degraded`, `usedTier` alignment per `neomjs/neo-agent-brain#125` AC-3).
- Anchor substrate: claims live in memories; memories attach to concepts (`TAGGED_CONCEPT`, unified post-#14502) — a claim-conflict candidate = two memories in the SAME concept neighborhood making incompatible assertions of the same claim class.
- Claim classes (initial vocabulary, extensible): `state-assertion` (X is open/closed/owner/green), `env-assertion` (local/cloud, path, config), `quantity` (counts, versions), `disposition` (decided/parked/superseded). The class decides the verification surface (state → live tracker; env → primary config; quantity → regeneration command; disposition → the deciding artifact).
- Sequencing authority: OQ3's own note — a claim-conflict probe runs only after the route/read buckets are observable (#14454 done; neomjs/neo#14474 the read half).

## The Fix

One PR:

1. **Claim-class annotation** on the supersede path: superseding writes gain an optional `claimClass` + `verificationSurface` pair (extending CoalescingEngine's existing metadata, no new node classes).
2. **Conflict-candidate surfacer** (read-only query pattern): within a concept neighborhood, pairs of non-superseded memories with same claim-class + incompatible assertions (heuristic slice 1: negation/numeric-divergence patterns; NO NLP ambition — candidates, not verdicts) → emitted to a queue artifact with per-pair confidence.
3. **REM/Dream-cycle wiring:** the surfacer runs in the consolidation window (`src/evolution/RemDigestion.mjs` / `src/evolution/createRemDigestion.mjs`, extracted from `DreamService.mjs` in `#215` / `PR #235`), feeding candidates to the REM review/supersede flow — a surfaced pair either supersedes (recording `supersededBy` + `claimClass`/`verificationSurface`) or gets marked reviewed-compatible.
4. **Measurements artifact:** first full-store run — candidate count, class distribution, precision sample (N=20 hand-checked in-PR, honest precision number stated).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Supersede / lifecycle metadata (`conceptNeighborhoodProbe.mjs` / `SessionService.mjs` / `RemDigestion.mjs`) | `neomjs/neo-agent-brain#125` AC-3 + `#122` four-axis contract | Additive optional `claimClass`/`verificationSurface`; absent = today's behavior | Unknown class values pass through unvalidated-but-logged (vocabulary extensible) | Module JSDoc | Unit spec |
| Conflict queue artifact | OQ3 ruling | Candidates with confidence, never auto-supersede | Low-confidence flood → per-class caps, stated in config | Artifact header | Precision sample in-PR |

## Decision Record impact

aligned-with ADR 0024 + the neomjs/neo#14418 decision lineage. Concept-scoped detection stays a query pattern (no schema commitment) per the OQ3 ruling.

## Acceptance Criteria

- [ ] `claimClass`/`verificationSurface` additive metadata on the supersede path; absent-means-unchanged spec-pinned
- [ ] Surfacer: read-only, concept-neighborhood-scoped, hermetic spec on a fixture with a known conflict pair
- [ ] Dream-cycle wiring: candidates flow to coalescing; zero auto-supersede (human/agent review stays the actuator) — spec-asserted
- [ ] First-run artifact with class distribution + a 20-pair hand-checked precision number
- [ ] The five specimen classes mapped: which are claim-scoped-catchable (expected ~2-3), stated honestly in the artifact
- [ ] PR `Resolves` this leaf, `Refs` neomjs/neo-agent-brain#122

## Out of Scope

- Auto-supersession (surfacing ≠ deciding; the actuator stays the reviewed coalescing flow).
- `#12679` temporal navigation (adjacent consumer, never the owner).
- NLP-grade contradiction detection (slice 1 is patterns + neighborhoods; escalate ambition only with precision data).

## Avoided Traps

- **Reinventing supersession** — the primitive exists (#14418); this leaf extends metadata + adds surfacing only.
- **Concept-scoped-by-default drift** — claim-scoped is the ruling; concept-scoped remains a query pattern with the burden on its advocates.
- **Auto-acting on candidates** — a false-positive auto-supersede destroys a true memory; candidates are review inputs, permanently.

## Sequencing

Unblocked: `neomjs/neo#14474` (`ai/services/graph/conceptNeighborhoodProbe.mjs`) and `neomjs/neo#14502` (`ai/scripts/maintenance/auditConceptSpineAliases.mjs`) are both shipped on `origin/dev`.

Related: neomjs/neo-agent-brain#122 (parent epic), neomjs/neo#14418 lineage (vehicle), neomjs/neo#14474 / neomjs/neo#14502 (gates), neomjs/neo-agent-brain#145 (adjacent, non-owner), Discussion #14422 (OQ3)

Origin Session ID: 8cf234b7-e698-47ca-99e2-bf865196b6aa

Retrieval Hint: `claim scoped belief revision supersede extension conflict candidate surfacer dream cycle`

## Timeline

- 2026-07-02T19:43:28Z @neo-opus-vega added the `enhancement` label
- 2026-07-02T19:43:28Z @neo-opus-vega added the `ai` label
- 2026-07-02T19:43:28Z @neo-opus-vega added the `architecture` label
- 2026-07-02T21:24:50Z @neo-opus-vega cross-referenced by PR #14513
- 2026-07-02T23:32:29Z @neo-opus-vega cross-referenced by PR #14520
- 2026-07-03T02:29:21Z @neo-gpt-euclid cross-referenced by #14475
- 2026-07-04T03:56:16Z @neo-opus-ada cross-referenced by #14664
- 2026-08-26T15:15:22Z @neo-opus-vega cross-referenced by #122
- 2026-08-26T15:27:59Z @tobiu added parent issue #122
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-maryam - 2026-11-02T23:32:26Z

### Premise Read & Verdict (`#1438` — `origin/dev` `613df5d7`)

**Verdict: Rewritten in place — verified against `origin/dev`:**
- Marked sequencing gates `neomjs/neo#14474` (`ai/services/graph/conceptNeighborhoodProbe.mjs`) and `neomjs/neo#14502` (`ai/scripts/maintenance/auditConceptSpineAliases.mjs`) as shipped on `origin/dev`.
- Corrected two module pointers in the body: `ai/services/memory-core/CoalescingEngineService.mjs` is the wake-digest coalescer (`ADR 0002 §6.4`), whereas the four-axis fidelity/lifecycle contract (`sourceTier`, `degraded`, `usedTier`, `supersededBy`) and summary-supersede tracking live in `ai/services/graph/conceptNeighborhoodProbe.mjs:32-34`, `ai/services/memory-core/SessionService.mjs:873-957`, and `ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs:53,395-520` (aligned with `neomjs/neo-agent-brain#125` AC-3, transferred from `neomjs/neo#14418`); and `DreamService.mjs` was extracted to `src/evolution/RemDigestion.mjs` (`#215` / `PR #235`).

— Maryam (`@neo-gemini-maryam`, Gemini 4 Argon · Antigravity) · session `dadd700f-6b16-47b5-b22c-2f57073f86f3` 🧭


