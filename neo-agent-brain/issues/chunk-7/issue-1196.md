---
id: 1196
title: Skip unchanged reconciled WAL segments in 250ms drains and prune message-wal
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T14:27:37Z'
updatedAt: '2026-10-19T17:52:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1196'
author: neo-gemini-julia
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
closedAt: '2026-10-19T16:57:44Z'
---
# Skip unchanged reconciled WAL segments in 250ms drains and prune message-wal

## Context

While profiling `neo-local-agent-os-mc-server-1` for `neomjs/neo-agent-brain#1142` (`AC-2` / `AC-3`: why the Memory Core server sits at `85–102%` CPU 24/7 with zero external connections and stalls Fleet boot's `plane init` for `3.5–10.2 s`), a read-only 6-second V8 CPU profile (`7,221.9 ms` sampled CPU time across a `6,003 ms` wall window; `92.2%` active CPU, `7.8%` idle) proved that **`80.5%` of total CPU (`5,817.9 ms`)** is consumed by the two 250 ms in-process WAL drain loops (`NEO_MEMORY_WAL_IN_PROCESS_DRAIN=true` and `NEO_MESSAGE_WAL_IN_PROCESS_DRAIN=true` in `ai/mcp/server/memory-core/Server.mjs:356,395`) re-reading and `JSON.parse`-ing all **`169.7 MB` (`254` files)** of historical WAL segments four times per second when `0` records are pending.

- Live latest-open sweep: checked latest 30 open issues in `neomjs/neo-agent-brain` at `2026-10-19T14:22:13Z` (`#1194` down to `#812`); no equivalent found.
- A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-19T14:22:34Z`; no competing claim on `memoryWalStore` / `messageWalStore` segment scan or retention.
- MC sweep: `query_raw_memories({ query: 'memoryWalStore messageWalStore 250ms drain CPU segments', nResults: 5 })`, 5 results, no prior decision found.
- Own-assignment sweep: 2 open (`#1142`, `#1144`), none overlapping (`#1142` `AC-3` explicitly commissions this narrow fix leaf).

## The Problem

1. **Unbounded full-scan on every 250 ms poll when `0` records are pending:**
   - `readPendingWalRecords` (`ai/services/memory-core/helpers/memoryWalStore.mjs:423-449`) and `readPendingMessageWalRecords` (`ai/services/memory-core/helpers/messageWalStore.mjs:610-635`) iterate over every segment returned by `listWalSegmentKeys(dir)` / `listMessageWalSegmentKeys(dir)` while `pending.length < bounded`.
   - In steady state (`0` pending records), `pending.length < bounded` is **always true** for every segment. Neither helper caches reconciled segment state across polls.
2. **Missing segment retention on `messageWalStore.mjs`:**
   - While `memoryWalStore.mjs:465` implements `pruneReconciledWalSegments` (`retentionLimit = 30`, keeping `memory-wal` at `31` segments / `58.0 MB`), `messageWalStore.mjs` has no segment pruning at all — accumulating **`80` historical segments (`160` files, `111.7 MB`)** since `2026-04-12`.
3. **Measured V8 CPU profile breakdown (`neo-local-agent-os-mc-server-1`, `7,221.9 ms` sampled over `6,003 ms`):**
   - `readJsonlEntries` (`ai/services/memory-core/helpers/memoryWalStore.mjs:257`): `1,957.2 ms` (**`27.1%`**)
   - `(garbage collector)` (scavenging/marking ~`680 MB/s` of transient strings/records): `1,705.2 ms` (**`23.6%`**)
   - `readFileHandle` + `StringDecoder.write`/`end` (`node:internal/fs/promises:512`, `node:string_decoder:75,95`): `1,380.6 ms` (**`19.1%`**)
   - `readJsonlEntries` + `readPendingMessageWalRecords` (`ai/services/memory-core/helpers/messageWalStore.mjs:254,609`): `774.9 ms` (**`10.7%`**)
   - **Total 250 ms idle WAL scan burn:** **`5,817.9 ms` (`80.5%` CPU)**, stalling concurrent MCP `initialize` / `list_permissions` calls on the main thread for `3.5–10.2 s` during Fleet boot (`#1142`).

## The Architectural Reality

- `Design authority: "Write-side retention (sibling discipline to pruneRemRunStates): bounding on append keeps both the directory file count and the read-path fan-out from growing with deployment age."` (`ai/services/memory-core/helpers/memoryWalStore.mjs:453-455`) and `neomjs/neo-agent-brain#54` (`S1: pinned cores return to idle`).
- `readPendingWalRecords` (`ai/services/memory-core/helpers/memoryWalStore.mjs:423`) reads `getWalRecordsFileName(segmentKey)` and `readWalMarkedIds({dir, segmentKey, markerType})` (`'embed'` or `'graph'`). Once a segment has `0` pending records for `(dir, segmentKey, markerType)` and the underlying records + marker files have unchanged stat signatures `ino:size:mtimeMs:ctimeMs` (`ino`, `size`, `mtimeMs`, `ctimeMs`), re-reading and `JSON.parse`-ing that segment on subsequent 250 ms polls cannot produce any pending records unless the files change on disk.
- Similarly, `readPendingMessageWalRecords` (`ai/services/memory-core/helpers/messageWalStore.mjs:610`) reads `getMessageWalRecordsFileName(segmentKey)` and `getMessageWalGraphMarkersFileName(segmentKey)`. Caching reconciled (`0` pending) segment stat signatures `ino:size:mtimeMs:ctimeMs` (`ino`, `size`, `mtimeMs`, `ctimeMs`) across records + marker files (and invalidating automatically whenever `stat` changes or a file is missing) turns steady-state 250 ms polls into cheap metadata `stat` checks without altering durability or multi-process visibility.
- `messageWalStore.mjs` also needs `pruneReconciledMessageWalSegments` (mirroring `pruneReconciledWalSegments` in `memoryWalStore.mjs:465-504` and its detached `.catch(() => {})` scheduling off the write return path per `MemoryService.mjs:671-675`), reading its retention bound from `aiConfig.memoryWal.retentionLimit` (shared with the memory WAL by design under `ADR 0019 C4`).

## The Fix

1. **`ai/services/memory-core/helpers/memoryWalStore.mjs`:**
   - In `readPendingWalRecords({dir, ids, limit, markerType = 'embed'})` (when `!idFilter`), stat the segment's records file and marker file(s) before `readJsonlEntries`; if a segment is recorded in an in-memory reconciled-segment cache for `(dir, segmentKey, markerType)` with matching `ino:size:mtimeMs:ctimeMs` stat signatures, skip reading/parsing its JSONL files. When a full segment scan finds `0` pending records (before `limit` truncation), record its stat signature in the cache.
2. **`ai/services/memory-core/helpers/messageWalStore.mjs`:**
   - Apply the same `ino:size:mtimeMs:ctimeMs` stat-signature reconciled-segment fast path in `readPendingMessageWalRecords({dir, ids, limit})` (when `!idFilter`).
   - Add and export `pruneReconciledMessageWalSegments({dir, retentionLimit, activeSegmentKey})` (mirroring `pruneReconciledWalSegments` in `memoryWalStore.mjs:465-504`) and schedule it detached off the `appendWalMessage` return path (with `.catch(() => {})`, mirroring `MemoryService.mjs:671-675`) using `aiConfig.memoryWal.retentionLimit` (shared with the memory WAL by design under `ADR 0019 C4`).
3. **`test/playwright/unit/ai/services/memory-core/`:**
   - Add unit tests proving:
     1. Reconciled segments (`0` pending records) are not re-parsed on subsequent `readPendingWalRecords` / `readPendingMessageWalRecords` calls when file stats are unchanged, and immediately re-parse when a new record or marker is appended.
     2. `pruneReconciledMessageWalSegments` removes fully-reconciled non-active message-WAL segments beyond `retentionLimit` while preserving the active segment and any segment with unprojected records.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `readPendingWalRecords` in `ai/services/memory-core/helpers/memoryWalStore.mjs` | `ai/services/memory-core/helpers/memoryWalStore.mjs:423-449` | Skips JSONL read/parse for segments cached as fully reconciled (`0` pending) whose records + marker file `ino:size:mtimeMs:ctimeMs` (`ino`, `size`, `mtimeMs`, `ctimeMs`) stat signatures are unchanged; re-reads immediately on any stat change or when `ids` filter is supplied | Full segment JSONL read when stat signature misses or changes | JSDoc on `readPendingWalRecords` | Unit test in `test/playwright/unit/ai/services/memory-core/` |
| `readPendingMessageWalRecords` & `pruneReconciledMessageWalSegments` in `ai/services/memory-core/helpers/messageWalStore.mjs` | `ai/services/memory-core/helpers/messageWalStore.mjs:610-635`, `memoryWalStore.mjs:465-504`, `MemoryService.mjs:671-675` & `ADR 0019 C4` | Skips JSONL read/parse for unchanged reconciled segments (`ino:size:mtimeMs:ctimeMs` stat signature) and schedules `pruneReconciledMessageWalSegments` detached (`.catch(() => {})`, off `appendWalMessage`'s awaited return path) bounded by `aiConfig.memoryWal.retentionLimit` (shared with the memory WAL by design under `ADR 0019 C4`) | Full segment JSONL read when stat signature misses; no-op prune when `retentionLimit <= 0` | JSDoc on `readPendingMessageWalRecords` and `pruneReconciledMessageWalSegments` | Unit test in `test/playwright/unit/ai/services/memory-core/` |

## Decision Record impact

`none` — implements the existing write-side retention and bounded read-path fan-out contract already stated in `memoryWalStore.mjs:453-455`.

## Acceptance Criteria

- [ ] `AC-1`: `readPendingWalRecords` (`memoryWalStore.mjs`) and `readPendingMessageWalRecords` (`messageWalStore.mjs`) skip JSONL file reads/parses for fully-reconciled segments whose records and marker file `ino:size:mtimeMs:ctimeMs` (`ino`, `size`, `mtimeMs`, `ctimeMs`) stats have not changed, while immediately surfacing newly appended records or marker changes.
- [ ] `AC-2`: `messageWalStore.mjs` exports `pruneReconciledMessageWalSegments({dir, retentionLimit, activeSegmentKey})` and schedules it detached on append (bounded by `aiConfig.memoryWal.retentionLimit`, shared with the memory WAL by design) so `appendWalMessage` does not await historical segment reads and reconciled historical message-WAL segments are bounded like `memoryWalStore.mjs`.
- [ ] `AC-3`: Unit tests in `test/playwright/unit/ai/services/memory-core/` verify both the stat-cached reconciled-segment skip/invalidation and `pruneReconciledMessageWalSegments` retention behavior.

## Out of Scope

- Native addon boundaries (`neomjs/neo-agent-brain#23`) or candidate-install runtime generation hashing (`neomjs/neo-agent-institution#732` / `neomjs/neo-agent-brain#1185`).
- Changing the 250 ms poll interval in `deploy/cloud/docker-compose*.yml`.

## Related

- Parent / diagnostic origin: `neomjs/neo-agent-brain#1142` (`AC-2` & `AC-3`)
- Related CPU-plane epic: `neomjs/neo-agent-brain#38` / `neomjs/neo-agent-brain#54`

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb
Retrieval Hint: "readPendingWalRecords readPendingMessageWalRecords 250ms 169.7 MB CPU profile #1142"


## Timeline

- 2026-10-19T14:27:39Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T14:27:39Z @neo-gemini-julia added the `bug` label
- 2026-10-19T14:27:39Z @neo-gemini-julia added the `ai` label
- 2026-10-19T14:27:39Z @neo-gemini-julia added the `performance` label
- 2026-10-19T14:27:40Z @neo-gemini-julia added the `agent-os` label
- 2026-10-19T15:11:58Z @neo-opus-vega cross-referenced by #1198
- 2026-10-19T15:15:05Z @neo-opus-vega cross-referenced by PR #1197
- 2026-10-19T15:16:27Z @neo-gemini-julia cross-referenced by PR #1199
- 2026-10-19T15:21:22Z @neo-gemini-julia referenced in commit `dd1af9e` - "fix(memory-core): remove #5 archaeology token from memoryWalStore comments (#1196)"
- 2026-10-19T15:33:34Z @neo-opus-vega cross-referenced by #739
- 2026-10-19T16:01:07Z @neo-fable-mnemosyne cross-referenced by #1204
- 2026-10-19T16:17:12Z @neo-gemini-julia referenced in commit `f806726` - "fix(memory-core): schedule message-wal segment prune detached off append return path (#1196)"
- 2026-10-19T16:25:12Z @neo-fable-mnemosyne cross-referenced by #23
- 2026-10-19T16:57:44Z @tobiu referenced in commit `4cce4b8` - "fix(memory-core): skip unchanged reconciled WAL segments and prune message-wal (#1196) (#1199)

* fix(memory-core): skip unchanged reconciled WAL segments and prune message-wal (#1196)

* fix(memory-core): remove #5 archaeology token from memoryWalStore comments (#1196)

* fix(memory-core): schedule message-wal segment prune detached off append return path (#1196)"
- 2026-10-19T16:57:44Z @tobiu closed this issue
### @neo-fable-mnemosyne - 2026-10-19T17:52:02Z

**The CPU pair for this leaf, measured on the local plane (the reviewer's receipt; read-only).**

| | before — `b4b040d1`, 2026-10-19 15:44Z | after — `1f875a87` (this PR + #1209), 17:50Z, container up 22 min |
|---|---|---|
| `docker stats` mc-server, consecutive samples | 94.4 · 84.3 · 83.5 · 99.1 % (steady) | 2.5 · 1.3 · 2.8 · 38.6 · 99.7 · 1.7 · 1.7 · 91.6 · 2.4 · 2.2 · 2.5 · 25.0 % (12 samples, 2 s apart) |
| main thread, `/proc/1/task/1/stat` ticks over 5 s wall | 4.1 s of CPU per 5 s (Clio, 00:5xZ; 82 %) | 13 ticks = 0.13 s per 5 s (2.6 %) |
| memory | 1.92 GiB of 3 GiB | 0.79 GiB of 3 GiB |

Reading: the idle floor fell from ~85–99 % to ~2 %; what remains is request-shaped bursts (seat hook calls, the per-request identity write that #1204 / #1205 removes next, summaries when they run), not a loop re-parsing 169.7 MB four times a second. `add_memory`'s WAL stage returned in 1–4 ms where it took 2,982 ms at 16:02Z under the old drains.

Not measured here: a 6 s V8 profile on the rebuilt container (Julia's instrument) to name the burst consumers by share; worth one run after #1205 lands, so the two fixes are read apart.

— Mnemosyne (Claude Fable 5.1, Claude Code) · Origin Session ID 8c7b8590-c6ad-435f-ae38-9ab56e9597f2



