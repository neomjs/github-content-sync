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
updatedAt: '2026-11-14T21:48:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1729'
author: neo-gemini-julia
commentsCount: 0
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
- This retirement leaf (blocked by `#1527`, sub-issue of `#122`): verifies the post-deploy receipt on the live plane and deletes the one-shot pass, its `SchemaVersion` constant, and its dedicated unit spec arm.

Ownership: self-assigned (`@neo-gemini-julia`) per `pull-request` §6.2; `good first issue` / `help wanted` omitted because this is an already-owned post-deploy migration retirement leaf requiring live-plane receipt verification.

## The Problem

Once PR #1728 deploys and `IssueIngestor#ingestIssueStates()` completes one pass on the live plane, `SchemaVersion` holds `id = 'issue_ingestor_legacy_anchor_reset_v1'` and every legacy anchor weight (`2`, `1.5`, `2.5`, decayed `> 1.0` scent, and `0.05` discounts) is normalized back to base. Keeping `normalizeLegacyAnchoredIssueEdgesOnce()` after that run would leave dead migration code in `IssueIngestor.mjs`.

## The Architectural Reality

- `ai/services/ingestion/IssueIngestor.mjs`: hosts `LEGACY_ANCHOR_RESET_MARKER_ID = 'issue_ingestor_legacy_anchor_reset_v1'` and `normalizeLegacyAnchoredIssueEdgesOnce()` (confirmed via `npm run ai:structure-map -- --files --loc`: `ai/services/ingestion/IssueIngestor.mjs` owns issue graph projection).
- `ai/graph/storage/SQLite.mjs`: `SchemaVersion` (`id TEXT PRIMARY KEY, version INTEGER NOT NULL, updated_at TEXT`) persists the one-shot marker independently of `GRAPH_SCHEMA_VERSION_ID` (`graph_schema`).
- `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs`: carries the legacy-anchor store normalization spec arm added for PR #1728 RA-1.

Decision Record impact: none.

## The Fix

1. After `#1527` (PR #1728) merges and deploys to the live plane, verify after at least one corpus-projection pass that:
   - `SchemaVersion` holds `issue_ingestor_legacy_anchor_reset_v1` (or live `get_neighbors` on `issue-20747` and `issue-20895` confirms `MESSAGE -> REFERENCES_TICKET` weights are `1` and `DISCUSSED_IN` carries undecayed/decayed base `<= 1` rather than `1.96`), and record the receipt on this ticket.
2. Delete `LEGACY_ANCHOR_RESET_MARKER_ID`, `normalizeLegacyAnchoredIssueEdgesOnce()`, its in-memory `WeakSet` fallback, and any helper used solely by that one-shot pass from `ai/services/ingestion/IssueIngestor.mjs`.
3. Delete the dedicated one-shot migration spec arm from `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs` while keeping the permanent `ANCHORED_ISSUE_EDGE_TYPES` and `computedGoldenPathRouting` emphasis specs intact.

## Acceptance Criteria

- [ ] AC-1: Live-plane deployment + execution receipt recorded on this ticket confirming `issue-20747` (`MESSAGE -> REFERENCES_TICKET` = `1`, `DISCUSSED_IN` stripped of `2.0x` anchor emphasis) and `issue-20895` (`MESSAGE -> REFERENCES_TICKET` = `1`, no `0.05` discount).
- [ ] AC-2: `normalizeLegacyAnchoredIssueEdgesOnce()`, `LEGACY_ANCHOR_RESET_MARKER_ID`, and any code used only by the one-shot pass are removed from `ai/services/ingestion/IssueIngestor.mjs`.
- [ ] AC-3: The one-shot migration spec arm in `test/playwright/unit/ai/services/ingestion/IssueIngestor.spec.mjs` is removed and the `IssueIngestor`, `GraphService`, and `GoldenPathSynthesizer` unit suites pass.

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

