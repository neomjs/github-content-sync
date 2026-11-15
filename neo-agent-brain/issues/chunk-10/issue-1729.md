---
id: 1729
title: 'Retire one-shot IssueIngestor legacy anchor weight reset after #1527 deploy'
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
  - tech-debt
assignees:
  - neo-gemini-julia
createdAt: '2026-11-14T21:48:17Z'
updatedAt: '2026-11-15T06:47:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1729'
author: neo-gemini-julia
commentsCount: 1
parentIssue: 122
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1527 The issue ingestor resets every edge of an open issue each pass'
blocking: []
---
# Retire one-shot IssueIngestor legacy anchor weight reset after #1527 deploy

## Context

Required by `pull-request` §6.2 (*One-shot logic*) and PR #1728 review [`PRR_kwDOUBzDFM8AAAABYBkpmw`](https://github.com/neomjs/neo-agent-brain/pull/1728#pullrequestreview-5907229083) (RA-1). On `origin/dev` (`e4bc5491`), `IssueIngestor`'s legacy Ancestral Anchor wrote emphasis (`1.5`, `2.0`, `2.5`) and open-blocker discounts (`0.05`) directly onto every edge incident to `OPEN` issues (measured live on `issue-20747` with `SESSION_SUMMARY -> DISCUSSED_IN` at `1.96` and eleven `MESSAGE -> REFERENCES_TICKET` at `2`, and `issue-20895` with three `MESSAGE -> REFERENCES_TICKET` at `0.05`).

PR #1728 (`#1527`) moves emphasis to Golden Path read time (`getComputedIssueEmphasisMultiplier`) and adds a one-shot normalization pass (`IssueIngestor#normalizeLegacyAnchoredIssueEdgesOnce`, gated by `SchemaVersion` key `issue_ingestor_legacy_anchor_reset_v1`) so the first post-deploy `IssueIngestor#ingestIssueStates()` run strips legacy anchor weights across the SQLite graph store without flattening subsequent multi-session `linkNodes` reinforcement.

**Convergence artifact (`/lead-role` §5 Guard B responsibility map):**
- `ai/services/ingestion/IssueIngestor.mjs` (`#1527` / PR #1728): ships `normalizeLegacyAnchoredIssueEdgesOnce()` + `SchemaVersion` receipt `issue_ingestor_legacy_anchor_reset_v1`.
- This retirement leaf (blocked by `#1527`, sub-issue of `#122`): verifies the post-deploy receipt on the live plane and deletes the one-shot pass, its `SchemaVersion` constant, and its dedicated unit spec arms.

Ownership: self-assigned (`@neo-gemini-julia`) per `pull-request` §6.2; `good first issue` / `help wanted` omitted because this is an already-owned post-deploy migration retirement leaf requiring live-plane receipt verification.

## The Problem

Once PR #1728 deploys and `IssueIngestor#ingestIssueStates()` completes one pass on the live plane, `SchemaVersion` holds `id = 'issue_ingestor_legacy_anchor_reset_v1'` and every legacy anchor weight (`2`, `1.5`, `2.5`, decayed `> 1.0` scent, and `0.05` discounts) is normalized back to base. Keeping `normalizeLegacyAnchoredIssueEdgesOnce()` after that run would leave dead migration code in `IssueIngestor.mjs`.

## The Architectural Reality

- `ai/services/ingestion/IssueIngestor.mjs`: hosts `LEGACY_ANCHOR_RESET_MARKER_ID = 'issue_ingestor_legacy_anchor_reset_v1'`, `resolveNormalizedLegacyIssueEdgeWeight()`, and `normalizeLegacyAnchoredIssueEdgesOnce()` (confirmed via `npm run ai:structure-map -- --files --loc`: `ai/services/ingestion/IssueIngestor.mjs` owns issue graph projection).
- `ai/graph/storage/SQLite.mjs`: `SchemaVersion` (`id TEXT PRIMARY KEY, version INTEGER NOT NULL, updated_at TEXT`) persists the one-shot marker independently of `GRAPH_SCHEMA_VERSION_ID` (`graph_schema`).
- `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs`: carries the legacy-anchor store normalization spec arm (`:946-1018`) added for PR #1728 RA-1.
- `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs`: imports `LEGACY_ANCHOR_RESET_MARKER_ID` (`:2366-2503`) for the one-shot SQLite normalization check alongside the permanent post-ingest `GraphService#linkNodes()` reinforcement + `2.0x` read-time emphasis assertion.

Decision Record impact: none.

## The Fix

1. After `#1527` (PR #1728) merges and deploys to the live plane, verify after at least one corpus-projection pass that:
   - `SchemaVersion` holds `issue_ingestor_legacy_anchor_reset_v1` (or live `get_neighbors` on `issue-20747` and `issue-20895` confirms `MESSAGE -> REFERENCES_TICKET` weights are `1` and `DISCUSSED_IN` carries undecayed/decayed base `<= 1` rather than `1.96`), and record the receipt on this ticket.
2. Delete `LEGACY_ANCHOR_RESET_MARKER_ID`, `resolveNormalizedLegacyIssueEdgeWeight()`, `normalizeLegacyAnchoredIssueEdgesOnce()`, its in-memory `WeakSet` fallback, and imports used solely by that one-shot pass from `ai/services/ingestion/IssueIngestor.mjs`.
3. Delete the dedicated one-shot migration spec arm from `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs` and remove `LEGACY_ANCHOR_RESET_MARKER_ID` + one-shot `SchemaVersion`/legacy-weight assertions from `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs` while keeping the permanent `ANCHORED_ISSUE_EDGE_TYPES`, SQLite `linkNodes` reinforcement survival, and `computedGoldenPathRouting` read-time emphasis specs intact.

## Acceptance Criteria

- [ ] AC-1: Live-plane deployment + execution receipt recorded on this ticket confirming `issue-20747` (`MESSAGE -> REFERENCES_TICKET` = `1`, `DISCUSSED_IN` stripped of `2.0x` anchor emphasis) and `issue-20895` (`MESSAGE -> REFERENCES_TICKET` = `1`, no `0.05` discount).
- [ ] AC-2: `normalizeLegacyAnchoredIssueEdgesOnce()`, `LEGACY_ANCHOR_RESET_MARKER_ID`, and any code used only by the one-shot pass are removed from `ai/services/ingestion/IssueIngestor.mjs`.
- [ ] AC-3: The one-shot migration spec arm in `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs` (and one-shot `LEGACY_ANCHOR_RESET_MARKER_ID` assertions in `GoldenPathSynthesizer.spec.mjs`) are removed while permanent reinforcement/emphasis assertions stay intact, and the `IssueIngestor`, `GraphService`, and `GoldenPathSynthesizer` unit suites pass.

## Out of Scope

- Changing `ANCHORED_ISSUE_EDGE_TYPES` (`PARENT_OF`, `BLOCKS`) or read-time `getComputedIssueEmphasisMultiplier()` introduced in `#1527`.
- Deleting the historical `issue_ingestor_legacy_anchor_reset_v1` row from the live SQLite `SchemaVersion` table (inert 1-row provenance).

## Avoided Traps

- **Ongoing clamp (`weight > 1.0 -> 1.0`) instead of a one-shot pass:** permanently flattens legitimate multi-session `GraphService#linkNodes()` reinforcement (`currentWeight + weight * 0.1` up to `5.0`) on `OPEN` issues on every corpus sync.
- **Deleting the one-shot pass before confirming the plane ran it:** would leave legacy `2.0` / `0.05` weights stranded in SQLite if merged before deployment ran an ingest pass.

## Related

- Blocked by #1527 (PR #1728)
- Parent epic: #122 (`21` existing sub-issues verified via GitHub `sub_issues` API)

Live latest-open + closed sweep: checked 156 open issues (latest 20 open + closed search `IssueIngestor baseWeight anchor`, Out of Scope / Avoided Traps read on hits, KB `query_documents` top hit `#1438` checked) at 2026-11-14T21:46Z; backlog band red (156 open), filed as required `pull-request` §6.2 one-shot retirement leaf under Epic #122; no equivalent or prior exclusion found.
A2A in-flight sweep: checked latest 30 `status: 'all'` messages at 2026-11-14T21:46Z; no overlapping claim.
MC sweep: `"neo-agent-brain IssueIngestor stored edge weights 1.96 0.05 REFERENCES_TICKET"`, 5 results, no prior decision found.
Own-assignment sweep: 2 open (`#1527`, `#1230` bodies read), `#1527` is the close-target blocker and `#1230` is unrelated (`Claude Desktop` preset store).

Origin Session ID: 2f084437-cc03-4b7b-aace-a27bff4aa3f9
Retrieval Hint: "retire normalizeLegacyAnchoredIssueEdgesOnce issue_ingestor_legacy_anchor_reset_v1 #1527"

## Timeline

- 2026-11-14T21:48:17Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-14T21:48:18Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T21:48:19Z @neo-gemini-julia added the `ai` label
- 2026-11-14T21:48:19Z @neo-gemini-julia added the `refactoring` label
- 2026-11-14T21:48:19Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T21:48:19Z @neo-gemini-julia added the `tech-debt` label
- 2026-11-14T21:50:15Z @neo-gemini-julia added parent issue #122
- 2026-11-14T21:50:17Z @neo-gemini-julia marked this issue as being blocked by #1527
- 2026-11-14T22:05:14Z @neo-gemini-julia cross-referenced by PR #1728
### @neo-gemini-julia - 2026-11-15T06:47:00Z

Lead-role active: planning, design dialogue, and peer coordination count as execution; suspend Auto Mode velocity bias until an exit condition is met.

## AC-1 Live-Plane Deployment & Execution Receipt (`2026-11-15T06:45Z`)

- **Live plane healthcheck (`neo-mjs-memory-core` `healthcheck({scope: 'plane'})`):**
  - `plane.id`: `neo-local-canonical`
  - `deployedRevision`: `6cfdf63e0069db818f3cd05a72efa092d82ee753` (includes `#1527` / PR #1728 merge `3f10dbe061502b9c3728f6ff47e327464bbceb2e` / `06e16078`).
- **`get_neighbors({id: 'issue-20747'})` (`2026-11-15T06:45:40Z`):**
  - `totalCount`: `11` (`truncated: false`)
  - `10` `MESSAGE -> REFERENCES_TICKET` edges: all `weight: 1` (legacy `2` stripped)
  - `1` `SESSION_SUMMARY -> DISCUSSED_IN` (`summary_c0558bc6-81bc-4d50-970d-4ad2b58571c3`): `weight: 1` (legacy `1.96` `2.0x` anchor emphasis stripped)
- **`get_neighbors({id: 'issue-20895'})` (`2026-11-15T06:45:58Z`):**
  - `totalCount`: `7` (`truncated: false`)
  - `3` `MESSAGE -> REFERENCES_TICKET` edges (`MESSAGE:0e8ea5a4`, `MESSAGE:789c383d`, `MESSAGE:901c6a75`): all `weight: 1` (zero `0.05` blocker discount)
  - `3` `BLOCKS` (`issue-20889`, `issue-20893`, `issue-20894`) + `1` `PARENT_OF` (`issue-20892`): all `weight: 1`

## Pre-Execution Intake Record (`ticket-intake` + `self-authored-carve.md`)

- **Blocker & Epic Gate:** `#1527` verified `CLOSED` (`2026-11-14T23:29:00Z`); Parent Epic #122 has independent `epic-review` comments on record ([#122#issuecomment-5427382441](https://github.com/neomjs/neo-agent-brain/issues/122#issuecomment-5427382441) by `@neo-gpt-euclid` and [#122#issuecomment-5427382584](https://github.com/neomjs/neo-agent-brain/issues/122#issuecomment-5427382584) by `@neo-gpt-emmy`).
- **Prescription & Core-Idiom Pre-Flight (`§1.9.6`):** Read `src/core/Base.mjs`, `src/Neo.mjs`, and `src/state/Provider.mjs`. `Prescription checked: ai/services/ingestion/IssueIngestor.mjs — owns the concern` (`normalizeLegacyAnchoredIssueEdgesOnce`, `resolveNormalizedLegacyIssueEdgeWeight`, `LEGACY_ANCHOR_RESET_MARKER_ID`, `normalizedInMemoryStores`, and sole-use imports live there; `GraphService.mjs` still consumes `syncCachedEdgeProperties` internally at `:1030` and `GoldenPathSynthesizer.mjs` consumes `computedGoldenPathRouting.mjs`).
- **Ticket Reality Classification:**
  - `Ticket age`: `createdAt: 2026-11-14T21:48:17Z`, `updatedAt: 2026-11-15T06:46Z` (body sharpened in place to include `GoldenPathSynthesizer.spec.mjs:2366-2503` `LEGACY_ANCHOR_RESET_MARKER_ID` cleanup).
  - `Bot stale-band`: `pre-stale` (`stale`: false, `no auto close`: false).
  - `Currency / successor-risk evidence`: `git log origin/dev --since="2026-11-14T21:46:00Z"` shows only `06e16078` (PR #1728) touched `IssueIngestor.mjs` / specs; zero competing PRs (`gh pr list --search "1729"`); KB `query_documents` top ticket hit is closed `#1527`.
  - **Verdict:** `valid-as-written` (positive ROI — eliminates dead migration code now that the live store is normalized).

- 2026-11-15T06:51:50Z @neo-gemini-julia cross-referenced by PR #1757
- 2026-11-15T07:53:33Z @neo-gemini-julia referenced in commit `fa5278e` - "feat(migrations): add upgrade CLI for legacy issue-edge weight reset (#1729)"

