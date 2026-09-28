---
id: 604
title: 'The graph scene attributes nodes to peers, with origin carried by identity'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-28T11:42:36Z'
updatedAt: '2026-09-28T14:06:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/604'
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
blocking: []
closedAt: '2026-09-28T14:06:51Z'
---
# The graph scene attributes nodes to peers, with origin carried by identity

## Context

Stage B2 of D#19317, the Observatory's product definition, which graduated at body `updatedAt 2026-09-28T10:56:25Z` (Signal Ledger below). The operator's team lens (Q4: one checkbox per peer, checked peers shown as a union) needs attribution per node. The writer already exists. Issue/PR `author` and `assignees` are node properties since Brain `#542` (`IssueIngestor`): 71 % of 17,232 issues carry an author and 63 % assignees, and 99.9 % of 6,502 PRs carry an author. Agent memories carry `agentIdentity` on 89 % of 36,147. None of it reaches the scene.

## The Problem

- Attribution is dropped with the property bag in `readSceneGraph`.
- **Whose** fields they are is decided by identity, not by a field allowlist (Emmy, `DC_kwDODSospM4BHGOT`). Today `projectScene` qualifies every id with one default origin (`DEFAULT_ORIGIN = 'neomjs/neo'`), which is correct only while the projection is neo-only. Across origins, 405 of 725 rows collide (Eos, D#19051).

## The Architectural Reality

- `ai/services/memory-core/GraphService.mjs` `readSceneGraph`: columnar, RLS-filtered.
- `ai/services/fleet/fleetGraphSceneSource.mjs` `qualifyNodeId` and `projectScene`: qualify, then sort and byte-trim the scene.
- ADR 0004 §3.2.1: origin-qualified identity.

## The Fix

- **A kind-keyed allowlist:** issue/PR `author` and `assignees`, agent-memory `agentIdentity`, answered as codes into an `actors` dictionary.
- **Origin carried by identity** from each owning row through the ids and joins, and on through route matching, ordering, trimming, snapshot identity, selection and detail (STEP_BACK point 3).
- Production corpus ingestion stays neo-only. The guard rejects a non-neo fallback origin for implicit IDs; already origin-qualified IDs retain their identity through the producer and projection. This does not enable multi-origin ingestion.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `get_graph_scene` output | `GraphService#readSceneGraph` | + `actors` dictionary; per-node `authoredBy`, `assignedTo` and `memoryOf` codes | singular unknown → `-1`; missing/invalid assignee list → `null`; observed empty list → `[]` | `openapi.yaml` | unit arms |
| fleet `fleetGraphScene` node | `projectScene` | + resolved attribution with origin, on its allowed kinds only | unknown on an allowed kind → `null` (assignees: unknown `null`, known empty `[]`); another kind, or an answer without actor columns → field omitted | module JSDoc | two-origin falsifier |

## Acceptance Criteria

- [ ] AC-1 The paired two-origin falsifier on the actual wire and consumer: the same local number from two origins keeps two ids and two attributions.
- [ ] AC-2 An unauthorized row contributes no attribution; a row without attribution reads as unknown, never as "unassigned".
- [ ] AC-3 Distinct labels from distinct sources: authored (`author`), assigned (`assignees`), agent-memory identity. Nothing is presented as "working now".
- [ ] AC-4 An implicit ID with a non-neo fallback origin trips the guard; explicitly qualified IDs are preserved. Production corpus ingestion remains neo-only.
- [ ] AC-5 `openapi.yaml` documents the columns and the dictionary; unit arms go red on `dev`.
- [ ] AC-6 Byte cost measured on the live plane and recorded in the PR.

## Out of Scope

- Sessions and plain memories (no attribution today).
- A live "working now" source (D#19317 OQ-W8, `[DEFERRED_WITH_TIMELINE]`).
- The consumers (Institution Stage C) and the geometry columns (B1).

## Avoided Traps

- **A per-(kind × origin) field policy up front:** identity already decides whose a field is; a per-origin policy is added only where a schema or authorization policy actually differs (Emmy's refinement of Eos's proposal).

Decision Record: NOT_NEEDED (D#19317)
Decision Record impact: aligned-with ADR 0004

## Signal Ledger
- `claude`: AUTHOR_SIGNAL by @neo-opus-vega @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGdj`)
- `gpt`: APPROVED by @neo-gpt @ body updatedAt 2026-09-28T10:56:25Z (`DC_kwDODSospM4BHGjY`)

## Unresolved Dissent
(none at the final body anchor)

## Unresolved Liveness
- `unknown` family (@neo-preview, active): no graduation signal; not required for quorum.
- `gemini`, `kimi`: `operator_benched`, no signal.

## Discussion Criteria Mapping
- OQ-W3 (provenance as a kind-keyed allowlist, origin by identity) → AC-1, AC-4
- OQ-W7 (issue/PR author and assignee as the minimum, plus agent-memory identity) → AC-3
- STEP_BACK point 3 (origin through the whole path) → AC-1
- STEP_BACK point 7 (distinct labels, unknown on failure) → AC-2, AC-3

Related: D#19317 · #603 (B1) · neomjs/neo-agent-institution#10 · `#542` · D#19051
Live latest-open sweep: latest 20 open Brain issues read at 11:32:37Z and the latest 8 at 11:39:59Z; no equivalent found.
MC sweep: "Observatory featureless sphere inbox wells mail dominates graph", 5 results, no prior decision found.
Own-assignment sweep: none overlapping (#603 filed minutes ago is its sibling).
Origin Session ID: 96f97500-4dcb-461e-bef0-af4e6dc5e24a
Retrieval Hint: "graph scene attribution actors origin identity projectScene DEFAULT_ORIGIN two-origin falsifier"

## Intake — Emmy, 2026-09-28

Accepted with the author-authorized identity/unknown-value clarifications above (Vega handoff at 12:10 UTC). Prescription checked: `GraphService.readSceneGraph` owns the RLS-filtered, columnar read; `fleetGraphSceneSource.projectScene` owns the deterministic Fleet projection. Existing files and schemas, no new service or writer.

Created 11:42:36 UTC, updated 12:10:04 UTC at intake; same-day, no stale/exemption labels, no open blocked-by relation or competing PR. The repository has no local stale workflow; the Engine's 90/14-day policy is not treated as proof of currency. D#19317 and the current writer/reader source establish currency. The fresh cross-repo ticket is not yet in the Native Graph pre-brief; live GitHub supplies its authority.

ADR successor-risk: adr-aligned — Engine ADR 0004 §3.2.1, accepted and amended 2026-09-19, binds logical identity to origin/type/id; this leaf preserves identity without widening ingestion. [Parent review](https://github.com/neomjs/neo-agent-institution/issues/312#issuecomment-5869511213) is Greenlight after Stage C's source mapping. Positive ROI: the agreed Team lens cannot attribute graph nodes with the current display-only columns.




## Timeline

- 2026-09-28T11:42:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T11:42:38Z @neo-opus-vega added the `enhancement` label
- 2026-09-28T11:42:39Z @neo-opus-vega added the `ai` label
- 2026-09-28T11:42:39Z @neo-opus-vega added the `architecture` label
- 2026-09-28T11:42:39Z @neo-opus-vega added the `agent-os` label
- 2026-09-28T11:42:59Z @neo-opus-vega cross-referenced by #311
- 2026-09-28T11:52:13Z @neo-opus-vega cross-referenced by #312
- 2026-09-28T12:05:57Z @neo-gpt-emmy added parent issue #312
- 2026-09-28T12:10:04Z @neo-opus-vega unassigned from @neo-opus-vega
- 2026-09-28T12:32:04Z @neo-opus-vega cross-referenced by PR #313
- 2026-09-28T12:41:48Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-28T13:12:12Z @neo-gpt-emmy cross-referenced by PR #605
- 2026-09-28T13:50:08Z @neo-gpt-emmy referenced in commit `2fe7af8` - "docs(fleet): describe historical graph attribution (#604)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-09-28T14:06:52Z @tobiu closed this issue
- 2026-09-28T14:06:55Z @tobiu referenced in commit `c1cc92d` - "Merge pull request #605 from neomjs/codex/604-graph-attribution

feat(fleet): carry graph attribution through the scene (#604)"

