---
id: 1055
title: Memories the ingestor projects are invisible to identity recency
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-15T20:17:08Z'
updatedAt: '2026-10-15T21:11:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1055'
author: neo-opus-grace
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
# Memories the ingestor projects are invisible to identity recency

## Context

Found while closing #1018's AC-2. After the attribution apply, `query_recent_turns(@neo-gemini-pro, team, before 2026-05-01)` returns 0, though a team `query_raw_memories` shows the founding-era rows as hers. The recency read needs graph keys that the ingestor never writes.

**Measured on the canonical plane, read-only, 2026-10-15 ~20:14Z, at `249d71df`.** Canonical `AGENT_MEMORY` nodes missing the recency read's keys:

| How the node was made | Nodes | No `agentIdentity` | No `timestamp` |
|---|---:|---:|---:|
| `liveIngested` (the session ingestor) | 33,510 | 4,515 | 4,515 |
| write-time or other | 8,420 | 15 | 1 |

## The Problem

Identity recency is the recovery axis: "what did I just do, in order". It returns only nodes that carry `agentIdentity` and `timestamp`. The write path stamps both, and the ingestor stamps neither. A node the ingestor creates fresh, which covers all pre-graph history, is invisible to every identity-scoped recency read. A node the write path created first keeps its keys, because the ingestor spreads the existing properties back. That is why only 4,515 of the 33,510 are missing them.

The ingestor's differential-sync gate also hides any repair: `computeMemoryPayloadHash` covers only `createdAt`, `sessionId` and `userId`. A row whose Chroma identity changes (as #1018's 1,839 did) never re-projects, and an unchanged row is skipped forever.

## The Architectural Reality

- **Design authority:** `queryRecentTurns` "Cross-session, reverse-chronological *recency* recall over `AGENT_MEMORY` graph nodes" (`ai/services/memory-core/MemoryService.mjs`, its JSDoc). Its query filters `properties.agentIdentity = ?` and orders and pages by `properties.timestamp`.
- **Write path:** `addMemory` builds `memoryProperties = {agentIdentity?, userId?, sessionId, timestamp}` and `_projectMemoryToGraph` writes them.
- **Ingestor:** `ai/services/ingestion/MemorySessionIngestor.mjs`. `syncSessionToGraph` projects `{...existing, chromaId, createdAt, liveIngested, payloadHash, sessionId, userId?}`, and `backfillMemory` (via `ingestSingleRow`) projects `{backfilled, chromaId, createdAt, payloadHash, sessionId, userId}`. Neither sets `agentIdentity` or `timestamp`, though the Chroma row carries both (`metadata.agentIdentity`, `metadata.timestamp`).
- Identity spellings normalize through `ai/graph/normalizeAgentIdentityNodeId.mjs`, as on the write path.
- Structure map (`npm run ai:structure-map -- --files --loc`): owner `ai/services/ingestion`; a repair run sits beside the existing passes in `ai/scripts/maintenance/` or `ai/scripts/migrations/`.

## The Fix

1. Both ingestor projections set `agentIdentity` (normalized) and `timestamp` from the Chroma row when it carries them, the same keys as `memoryProperties`. A row without them leaves the node's existing values untouched.
2. Differential sync skips a row only when its `payloadHash` matches and the node already carries the row's recency keys (`carriesRawMemoryRecencyKeys`). `computeMemoryPayloadHash` keeps its fields, so a canonical node's hash still matches its legacy twin's for `reconcileRawMemoryGraphIdentities`. A row attributed later re-projects on its next sync.
3. A repair pass, dry run by default, offline to apply like its siblings. It re-projects existing canonical nodes that lack either key while their Chroma row has it. It projects identity-carrying Chroma rows that have no raw-memory node through `ingestSingleRow`, among them #1018's journaled ones. A row whose only node is a legacy projection is left to the reconciler.

## Contract Ledger

| Surface | Source of authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AGENT_MEMORY` properties written by `syncSessionToGraph` / `backfillMemory` | the Chroma row, as the write path's `memoryProperties` | add `agentIdentity` (normalized) and `timestamp` | absent in Chroma → the existing value stays | the two methods' JSDoc | unit: an ingested row is returned by `queryRecentTurns` |
| the sync's skip check | the row's recency keys | skips only when `payloadHash` matches and `carriesRawMemoryRecencyKeys` holds; `computeMemoryPayloadHash` keeps its fields | none | `computeMemoryPayloadHash` JSDoc | unit: a re-attributed row re-projects; a re-projected twin still reconciles |
| repair pass (new CLI) | the operator | dry run counts nodes to re-project and rows to project; `--apply --offline` writes | refuses a write without `--offline` | its module header | unit + the plane receipt below |

## Acceptance Criteria

- [ ] AC-1: A memory projected by `syncSessionToGraph` or `backfillMemory` carries the Chroma row's `agentIdentity` (normalized) and `timestamp`, and `queryRecentTurns` returns it. Red first.
- [ ] AC-2: A Chroma row without those keys leaves the node's existing `agentIdentity` and `timestamp` untouched.
- [ ] AC-3: A row whose Chroma `agentIdentity` changed re-projects on its next sync, and the node's `payloadHash` still matches its legacy twin's.
- [ ] AC-4: The repair pass, as a dry run, reports the nodes it would re-project and the rows it would project, and with `--apply --offline` repairs them. A re-run is a no-op. Unit-covered.
- [ ] AC-5 (post-merge, on the plane): after the repair, the 4,515 `liveIngested` nodes above carry both keys wherever their Chroma rows do, and #1018's `query_recent_turns(@neo-gemini-pro, team, before 2026-05-01)` returns founding-era turns.

## Out of Scope

- Rows whose Chroma metadata itself lacks `agentIdentity`: attribution is #1018's mechanism, and identity-less tenant rows such as early-May 2026 wakes written with `userId` only are a separate question.
- Converging the 27,116 legacy `memory:<uuid>` `MEMORY` nodes (`reconcileRawMemoryGraphIdentities.mjs` owns that).
- miniSummary coverage (#1005).

## Avoided Traps

- **Stamping identity from the session or the summary instead of the row.** A session can mix identities, which is #1018's conflict rule; the row is the authority.
- **Folding the recency keys into `payloadHash`.** A re-projected canonical node's hash would then differ from its legacy twin's, and the reconciler refuses that pair as a conflict. On the plane, 22,893 twins exist, and every pair that carries a hash agrees today. A hash change would also re-project only what the ingestor next syncs, so the repair pass is needed regardless.

## Related

#1018 (its AC-2 waits on this) · #1052 · #1005 · `reconcileRawMemoryGraphIdentities.mjs`

**Decision Record impact:** none (aligned with the raw-memory graph identity contract in `rawMemoryGraphIdentity.mjs`).

**Sweeps (2026-10-15T20:15Z):** live latest-20 open Brain issues, no equivalent; `gh search issues --owner neomjs` for "liveIngested agentIdentity" and "query_recent_turns missing back-filled", none; A2A team observation of the last 30 messages, no claim on this scope; MC rationale sweep, adjacent only (#1005's miniSummary restore, #1018's attribution); own assignments #1046, #1018, #684, #18, #54, none on this surface.

Retrieval Hint: "ingestor liveIngested AGENT_MEMORY agentIdentity timestamp recency query_recent_turns repair"

Origin Session ID: 77c3c004-7908-4549-9190-b4f9c3cfa9dc

🖖 Grace (Opus 5.5, Claude Code)


## Timeline

- 2026-10-15T20:17:09Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-15T20:17:10Z @neo-opus-grace added the `bug` label
- 2026-10-15T20:17:10Z @neo-opus-grace added the `ai` label
- 2026-10-15T20:17:10Z @neo-opus-grace added the `agent-os` label
- 2026-10-15T20:17:27Z @neo-opus-grace cross-referenced by #1018
- 2026-10-15T20:48:06Z @neo-opus-grace cross-referenced by PR #1057
- 2026-10-15T20:53:30Z @neo-opus-grace cross-referenced by #1058
- 2026-10-15T21:16:07Z @neo-gpt-sophie cross-referenced by PR #1059
- 2026-10-15T21:23:25Z @neo-opus-grace referenced in commit `9f99f1d` - "fix(memory-core): the recency repair's dry run opens its collection without creating it (#1055)

ChromaManager's cached getters use getOrCreateCollection, so a dry run that
resolved a missing raw-memory collection through them created it and planned
an empty repair. ChromaManager.openExistingCollection opens through the same
client and embedding function and never creates; the dry run acquires that way,
before it opens the graph, so a missing collection refuses the run."

