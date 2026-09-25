---
id: 511
title: The REM GC deletes durable records and the vectors of nodes it keeps
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T20:57:10Z'
updatedAt: '2026-09-25T21:44:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/511'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-09-25T21:31:14Z'
---
# The REM GC deletes durable records and the vectors of nodes it keeps

## Context

I found this while tracing #506. Every REM cycle ends in `GraphMaintenanceService.runGarbageCollection` (`src/evolution/RemDigestion.mjs:945`). #507 fixed its edge pass; the orphan pass after it is unchanged and still deletes data. The local plane's dream stays braked (`NEO_ORCHESTRATOR_DREAM_INTERVAL_MS=0`, @neo-opus-vega, 20:52Z) until this lands.

**Observed** on the local plane between 20:50Z and 21:10Z, from read-only probes of the graph SQLite and Chroma.

Session summaries in `neo-agent-sessions`, the collection `query_summaries` searches:

| `SESSION_SUMMARY` nodes | vector present | vector gone |
|---|---|---|
| with an edge | 286 | **0** |
| edgeless | 154 | **1,248** |

All 1,688 record `semanticVectorId` equal to their own id, so every one was embedded. Of the collection's deleters, the orphan pass is the only one keyed on edgelessness. The others:
- `purge_session` works per session.
- The embed drain's compensations cover memory records.
- The temporal pruner writes `neo-temporal-summary`.

The orphan pass's own query at 21:10Z returned about 126k nodes. Among them are durable records the pass has no business forgetting:

| label | edgeless | what it is |
|---|---|---|
| `AGENT_MEMORY` | 29,671 | the current raw-memory label (`RAW_MEMORY_NODE_LABEL`); only the legacy `MEMORY` is protected |
| `MESSAGE` | 3,362 | mailbox records. At 19:51Z the pass removed 2,264 of them from storage. |
| `SESSION_SUMMARY` | 1,402 | the table above |
| `nl-transaction-archive` | 20 | the Neural Link transaction record, stored as an edgeless node by design |
| `SYSTEM_CLOCK` | 1 | `_SYSTEM_STATE`, whose `lastDecayedAt` is the decay's 24-hour lock |
| `KnowledgeBaseTenantManifest` | 1 | a KB tenant's ingestion state, written edgeless (`IngestionService.mjs:1360`) |

**The decay lock.** The dream log shows `Running ambient topology decay` at 17:10, 17:19, 17:28 and 17:39Z on 09-24, and at 18:00 and 18:09Z today. The lock allows one run per 24 hours.
- Observed: each cycle runs the GC (inside `processUndigestedSessions`) before its finalization decay (`RemDigestion.mjs:1194`). A missing clock is re-created without `lastDecayedAt`, which reads as 0 (`GraphService.mjs:968-981`).
- Observed: `upsertNode` lazy-loads the stored row before merging (`GraphService.mjs:300`), so a cache miss alone keeps `lastDecayedAt`. Only a storage delete loses it, and for an edgeless node that delete is the orphan pass's, on a clock the previous decay left cached.
- This fits every line in the log: back-to-back cycles in one process decay each time, and a cycle whose GC found the clock uncached (19:51Z, 20:46Z) skips.

## The Problem

Three defects stack in one pass.

1. **The protected labels lag the label set.** `getOrphanedNodes` forgets every label it doesn't name. The list missed `SESSION_SUMMARY` (whose node id is its vector id), the current raw-memory label after the `MEMORY` → `AGENT_MEMORY` rename, `MESSAGE` (whose edges `PROTECTED_EDGE_TYPES` already keeps as records), and three records written edgeless by construction.
2. **The purge covers more ids than the delete.** The purge deletes vectors for every orphan id. `GraphService.removeNodes` goes through `Database.removeNode`, which reaches SQLite only through the cache's mutate event, so only cached orphans leave storage. The result is a node kept with its vector gone, where `neomjs/neo#9740` intended both to leave together.
3. **Failures are swallowed.** Both collection deletes end in `.catch(() => {})`, so a failed purge would pass silently. Today's purges landed.
   - Observed: the summary sweep replays every missing summary from its stored `SummarizationJobs` envelope (`sessionSummaryReceiptStore`). It replayed 1,394 summaries at 17:17–17:22Z and 1,397 at 18:10–18:14Z, each right after a cycle's purge.
   - Observed: at 20:50:51Z it replayed 154 before the orchestrator stopped at 20:51Z, and 1,402 − 154 = 1,248 are still missing.
   - So every cycle deletes roughly 1,400 summary vectors that `query_summaries` loses until the next sweep restores them. The loop costs a heavy-maintenance slot each time. 1,243 of the 1,248 have an envelope; 5 have no job row.

## The Architectural Reality

- `ai/services/memory-core/GraphService.mjs`: `getOrphanedNodes` (the protected-label chain) and `removeNodes`.
- `ai/graph/Database.mjs`: `removeNode` works on the cache only; storage changes only via `onNodesMutate`.
- `ai/services/graph/GraphMaintenanceService.mjs`: the orphan pass and the vector purge.
- `ai/services/memory-core/helpers/rawMemoryGraphIdentity.mjs`: `RAW_MEMORY_NODE_LABEL = 'AGENT_MEMORY'` and `LEGACY_RAW_MEMORY_NODE_LABEL = 'MEMORY'`.
- `neo-native-graph` holds 501 vectors: ADR 38, ISSUE 154, DISCUSSION 307, CLASS 2. Every label except CLASS is protected, so that collection lost nothing.

## The Fix

1. Hold the protected labels in one `Set` instead of the 13-clause chain. Add `SESSION_SUMMARY`, both raw-memory constants, `MESSAGE`, `SYSTEM_CLOCK`, `KnowledgeBaseTenantManifest` and `nl-transaction-archive`, each with its reason in the JSDoc.
2. Purge vectors only for the orphan ids that storage no longer holds after `removeNodes`.
3. Drop both `.catch(() => {})`, so a failed purge reaches the pass's existing warn.

## Acceptance Criteria

- [ ] `getOrphanedNodes` does not return an edgeless `SESSION_SUMMARY` (red on `dev`).
- [ ] An orphan that `removeNodes` leaves in storage keeps its vectors, and a removed orphan's vectors are purged (red on `dev`).
- [ ] The pass logs a failed collection delete.
- [ ] Post-merge: on the plane recreated with this head, one dream cycle's apoptosis log names how many orphans left storage and how many vector ids were purged. `neo-agent-sessions`' count does not drop across that cycle, and the summary sweep after it replays nothing.
- [ ] `getOrphanedNodes` does not return an edgeless `AGENT_MEMORY`, `MESSAGE`, `SYSTEM_CLOCK`, `KnowledgeBaseTenantManifest` or `nl-transaction-archive` node, and the pass leaves each in storage (red on `dev`).

## Out of Scope

- **Inverting the protected list into an allowlist of labels designed to be forgotten.** A denylist fails open: every new durable label is garbage until someone names it, and this ticket found six. Which labels are forgettable is a design call: #516.
- **`GraphService.removeNodes` no-ops for uncached ids in every caller.** Making it storage-deleting before the plane's edges are restored would turn the next orphan pass into a delete of about 126k nodes. That is #517, blocked by #516.
- **Restoring the 1,248 summaries.** The existing receipt replay does it on the next summary sweep with no model call, and once this lands they stay. The 5 without a job row need a re-summarization.
- **Batching the Chroma deletes.** After fix 1, CLASS (2 vectors) is the only embedded label an orphan can carry, and fix 3 makes any failure visible.

## Avoided Traps

- **Deleting every orphan from storage now,** which is #9740's full intent. On this plane the orphan set is mostly #506's victims.
- **Anchoring each summary with a new edge instead of protecting the label.** That adds a write path to fix a read-side policy; the protected list is where durable labels already live.

## Related

#506 · #507 · #509 · #64 · design origin `neomjs/neo#9740`, `neomjs/neo#9912`

Decision Record impact: none.
Live latest-open sweep: checked latest 20 open issues at 2026-09-25T20:56:06Z; no equivalent found (#509 restores receipts).
A2A in-flight sweep: last 30 messages; no claim on this scope.
MC sweep: "GraphMaintenanceService apoptosis orphaned nodes removeNodes uncached vectors purged" (MC degraded, retried) and "query_summaries misses sessions, session summary vectors missing, SESSION_SUMMARY orphan protected labels", 6 results, no prior decision found.
Own-assignment sweep: 4 open, none overlapping.
Structure map: `ai/services/graph` owns `GraphMaintenanceService.mjs`; no new file.

Revised 21:20Z: the durable-label rows, the decay-lock evidence and the fifth AC were added after the label census, in the same PR.
Corrected 21:30Z: an earlier revision said the 20:46Z purge "did not land" because 154 edgeless summaries still had vectors. The replay log shows those 154 were restored after it, so that claim and AC-4's "154 keep their vectors" are replaced above.

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201
Retrieval Hint: "REM GC orphan pass durable labels SESSION_SUMMARY AGENT_MEMORY decay clock"


## Timeline

- 2026-09-25T20:57:10Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T20:57:11Z @neo-opus-grace added the `bug` label
- 2026-09-25T20:57:12Z @neo-opus-grace added the `ai` label
- 2026-09-25T20:57:12Z @neo-opus-grace added the `agent-os` label
- 2026-09-25T20:59:03Z @neo-opus-vega cross-referenced by #506
- 2026-09-25T21:03:43Z @tobiu referenced in commit `8cc1cc4` - "test(graph): the session-summary arm asserts the orphan query itself skips the label (#511)"
- 2026-09-25T21:06:27Z @tobiu referenced in commit `90d0f48` - "style(graph): align the GC pass's opening declarations (#511)"
- 2026-09-25T21:06:30Z @neo-opus-grace cross-referenced by PR #515
- 2026-09-25T21:07:39Z @neo-opus-grace cross-referenced by PR #507
- 2026-09-25T21:15:37Z @tobiu referenced in commit `deada39` - "fix(graph): the orphan pass keeps raw memories, messages, the decay clock, KB manifests and NL archives (#511)"
- 2026-09-25T21:16:14Z @neo-opus-grace changed title from **The REM GC purges session-summary vectors of nodes it keeps** to **The REM GC deletes durable records and the vectors of nodes it keeps**
- 2026-09-25T21:20:34Z @neo-opus-grace cross-referenced by #516
- 2026-09-25T21:20:44Z @neo-opus-grace cross-referenced by #517
- 2026-09-25T21:31:14Z @tobiu referenced in commit `cd74d13` - "Merge pull request #515 from neomjs/grace/511-orphan-pass-summaries

fix(graph): the REM GC keeps durable records and purges a vector only with its node (#511)"
- 2026-09-25T21:31:14Z @tobiu closed this issue
### @neo-opus-grace - 2026-09-25T21:44:03Z

Recovery receipt, after the merge (read-only, 21:44Z). The plane runs cd74d13 since 21:36Z. Since then the summary sweep has replayed exactly **1,243** summaries from their `SummarizationJobs` envelopes ("Replayed durable session summary" in the dream-side log), and `neo-agent-sessions` holds **4,044** vectors, up from 2,799 (+1,243, +2 new sessions). So the envelope set is restored with no model call; the 5 summaries without a job row still need re-summarizing. AC-4 follows from the first guarded GC, which is running: the count must not drop across it, and the next sweep must replay nothing.

- 2026-09-25T21:46:00Z @neo-opus-vega cross-referenced by #64
- 2026-09-25T21:51:49Z @neo-opus-vega cross-referenced by PR #520
- 2026-09-25T21:52:38Z @neo-opus-vega cross-referenced by #521

