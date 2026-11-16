---
id: 1795
title: The vector-coverage audit counts Chroma's queue tail as missing vectors
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-16T08:24:51Z'
updatedAt: '2026-11-16T08:24:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1795'
author: neo-opus-vega
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
blocking:
  - '[ ] 1432 No defrag heal operation runs the KB''s store-bloat diagnosis'
---
# The vector-coverage audit counts Chroma's queue tail as missing vectors

## Context

The Brain #66 experiment ran on 2026-11-16 (receipt: https://github.com/neomjs/neo-agent-brain/issues/66#issuecomment-6911009482). It wrote 4,321, 7,321 and 62,654 rows into throwaway `chromadb/chroma:1.5.9` containers, the plane's image, and read the store at each stop:

- The HNSW vector segment persists every 1,000 rows. Its `max_seq_id` read 4,000, 7,000 and 62,000.
- The metadata segment holds every row: 4,321, 7,321 and 62,654.
- `embeddings_queue` keeps exactly the tail between the two (322 and 655 rows; the queue's lowest `seq_id` equals the vector segment's `max_seq_id`), and Chroma replays it at startup. Even after a SIGKILL, no row was lost.

So at any moment, up to 999 rows per collection are durable in the queue but absent from the persisted HNSW id map. That is Chroma's design, not damage.

## The Problem

`auditChromaVectorCoverage` (`ai/scripts/maintenance/checkChromaIntegrity.mjs:459`) compares two id lists through `compareMetadataToVectorIds` (`:422`):
- the metadata segment's ids, from `readMetadataEmbeddingIds`;
- the persisted HNSW id map in `index_metadata.pickle`, from `readVectorIndexIds`.

It never reads `embeddings_queue` or the vector segment's `max_seq_id`. Every row in the queue tail is therefore reported as `missingFromVector`, on every collection whose size is not a multiple of 1,000, on every run.

The output feeds more than the CLI:
- `ai/daemons/orchestrator/Orchestrator.mjs:53` imports the audit for the data-integrity coverage gatherer.
- `ai/daemons/orchestrator/services/dataIntegrityModeClassifier.mjs:113-115` reads `missingFromVectorCount > 0` with the documents present as a re-embeddable stall, and `Orchestrator.mjs:1034` hands `missingVectorIds` to the repair path.

So once coverage runs, healthy tail rows look like a stall that needs a re-embed, on every sweep. It is latent today: #1432 records that the cloud orchestrator mounts no Chroma data, so coverage has no recorded run on this plane. The mount work there would switch this on.

## The Architectural Reality

- `embeddings_queue` rows carry `seq_id`, `topic` (`persistent://default/default/<collection-uuid>`), `operation` and `id`. The #66 body records operation `3` as a DELETE. Read the other codes from the image's own schema, never from memory.
- `max_seq_id` maps each segment id to its persisted sequence. The vector segment's row there is the point the tail starts from.
- The audit already reads a copied SQLite snapshot through an injectable `execFn` (`sqlite3 -json`), so a fixture can drive every branch without a live store.

## The Fix

In `auditChromaVectorCoverage`, read each collection's queue tail from the same snapshot: the `embeddings_queue` rows on that collection's topic whose `seq_id` is above the vector segment's `max_seq_id`, in `seq_id` order. Apply their net effect per id: an add, update or upsert makes the id present, and a delete makes it absent. Do this before the comparison. Report the tail beside the drift, as `pendingInQueueCount` plus a sample in the same shape as the other samples, so a reader sees the pending rows instead of losing them.

## Acceptance Criteria

- [ ] AC-1: A row whose only vector is in the queue tail is not counted in `missingFromVectorCount` or `missingVectorIds`. It appears in `pendingInQueueCount` and its sample.
- [ ] AC-2: A tail DELETE of an id the HNSW map still holds is applied too, so that id is neither missing nor extra.
- [ ] AC-3: Unit tests use a fixture snapshot through `execFn` and prove both directions: a tail-only row counts as present, and a row with no vector and no queue entry still counts as missing.
- [ ] AC-4: The CLI's coverage output prints the pending count beside the drift counts.

## Out of Scope

- Mounting Chroma data into the cloud orchestrator, or un-blinding the coverage gatherer (#1432).
- Changing what the classifier or the repair path does with a real coverage gap.

## Avoided Traps

- **Treating `max_seq_id` gaps as damage.** The gap between the vector and metadata segments is Chroma's 1,000-row persist cadence, measured on the plane's own image.
- **A tolerance threshold (say, ignore under 1,000).** It would hide a real gap of the same size; the queue says exactly which ids are pending.

## Related

- #66 (the measurement), #1432 and #1434 (the gatherer this evidence feeds), #133 (the store-bloat roadmap).

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked the latest 20 open + open and closed search (`checkChromaIntegrity`, `embeddings_queue`, `vector coverage`, `auditChromaVectorCoverage`, `missingVectorIds`) at 2026-11-16T08:24Z. No equivalent found. Same-surface hits #1432 and #1434 un-blind the gatherer and do not cover the comparison; #133 is the bloat roadmap. Backlog band: 157 open (red); no owning ticket covers this comparison. A2A in-flight claim sweep (all read-states, last 30): none on this scope. MC sweep: `query_raw_memories("Chroma vector coverage audit reports missing vector ids that are durable in the embeddings queue tail")` surfaced the audit's origin (PR #13559, June), with no record of the queue tail. Own-assignment sweep: #1432 and #1434 are mine and read. Structure map: no new file; owner `ai/scripts/maintenance/checkChromaIntegrity.mjs`, consumer `ai/daemons/orchestrator/Orchestrator.mjs`.

Origin Session ID: 400a3843-07c2-462c-aa27-1bef77f253e4

Retrieval Hint: "vector coverage audit embeddings_queue tail max_seq_id sync_threshold missingFromVector chroma 1.5.9"

## Timeline

- 2026-11-16T08:24:53Z @neo-opus-vega added the `bug` label
- 2026-11-16T08:24:53Z @neo-opus-vega added the `good first issue` label
- 2026-11-16T08:24:53Z @neo-opus-vega added the `help wanted` label
- 2026-11-16T08:24:53Z @neo-opus-vega added the `ai` label
- 2026-11-16T08:24:54Z @neo-opus-vega added the `agent-os` label
- 2026-11-16T08:24:57Z @neo-opus-vega marked this issue as blocking #1432

