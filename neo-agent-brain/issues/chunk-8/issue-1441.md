---
id: 1441
title: Drop stored documents from existing turn rows and measure the reclaim
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-02T23:28:48Z'
updatedAt: '2026-11-02T23:28:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1441'
author: neo-opus-ada
commentsCount: 0
parentIssue: 133
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 128 Field↔document de-dup (~910MB) — consumer-aware refactor to one canonical representation'
blocking: []
---
# Drop stored documents from existing turn rows and measure the reclaim

## Context

This is scope items 3 and 4 of #128, split out on 2026-11-02 so that #128's write-side PR can resolve #128 on its own. It is a sub of #133.

Once #128 lands, new turn rows store their text once. Every turn row written before that still stores it twice:
- as the split metadata fields (`prompt`, `thought`, `response`);
- as the Chroma document, which also feeds Chroma's full-text index.

The last measurements are ~910 MB of duplicated turn text (2026-06-27) and ~556 MB of full-text index (2026-08-25). Neither has been re-measured. The shared store's `chroma.sqlite3` is 4.47 GB (#1432).

## The Problem

The bytes come back only when each existing turn row stops holding its document and a VACUUM returns the freed pages. Three constraints make this a governed migration rather than a loop:

1. **No safe in-place clear.**
   - chromadb 3.5.0 (`prepareRecords`) embeds any `documents` passed without `embeddings`, on `update` as on `add`. So `update({ids, documents: ['']})` would overwrite each turn's vector with the embedding of an empty string.
   - `validateDocuments` rejects `null`.
   - So a clear must pass the row's preserved vector with it, or the row must be re-created: `delete`, then `add({ids, embeddings, metadatas})`.
2. **No full copy.**
   - ADR 0027 §2.4 requires snapshot-before-mutate.
   - A 4.47 GB copy does not fit the guest disk (5.6 GiB free on 11-02, #1432).
   - Memory Core shadow promotion is disabled (the `repairMemoryCoreStoredEmbeddings.mjs` header).
   - So the snapshot is per slice: each row's prior document.
3. **The live file uses a rollback journal** (#1432). Measurements run on a copy of the file, never against the live store.

## The Architectural Reality

- **Turn rows** are memory-collection rows with `metadata.type === 'agent-interaction'`. `resolveTurnDocumentForRead` (`ai/services/memory-core/helpers/turnDocumentText.mjs`) reconstructs any turn whose stored document is empty or absent, and returns `null` for every other type.
- **Plan rows:** the memory collection also holds `implementation_plan` rows, written by `SessionService`. Their document is their content, and they keep it.
- **The qualifying test:** `composeTurnDocumentText(metadata)` is the derivation. A row qualifies only where it reproduces the stored document byte for byte.
- **Placement:** Chroma row backfills live in `ai/scripts/migrations/`, beside `backfillChromaSharedUserId.mjs` and `repairStaleEmbeddings.mjs` (the structure map at `dev` `613df5d7`).

## The Fix

1. **Choose the removal mechanism on a scratch store.** The candidates:
   - an in-place `update` to an empty-string document, passing the preserved vector;
   - `delete` followed by `add`, with no documents.

   The deciding measurements:
   - the vector is byte-identical afterwards;
   - the row's full-text index entry is gone;
   - a crash mid-batch leaves every row in the old shape or the new one, never absent.
2. **A resumable migration in `ai/scripts/migrations/`.** It:
   - selects turn rows whose stored document equals `composeTurnDocumentText(metadata)`;
   - snapshots each row's prior document before mutating it;
   - applies the chosen mechanism in batches;
   - skips rows that already have no stored document, which makes it idempotent.
3. **Reclaim, measured.** Measure the bytes of `chroma.sqlite3` and of the full-text index before and after, on a copy. Then run a VACUUM, coordinated with #1432.

## Acceptance Criteria

- [ ] The removal mechanism is chosen on a scratch store, and the three deciding measurements are recorded here.
- [ ] The migration drops a turn row's stored document only where `composeTurnDocumentText(metadata)` reproduces it byte for byte. It leaves every other row untouched and keeps each turn's vector byte-identical.
- [ ] It snapshots each row's prior document before mutating it, restores from that snapshot, and is idempotent and resumable.
- [ ] *(post-merge, operator-run)* It runs against the plane with the operator's yes. The reclaimed bytes of `chroma.sqlite3` and the full-text index are measured and recorded here.
- [ ] *(post-merge)* Memory search, session summarization and turn display show no regression on migrated rows.

## Out of Scope

- **The write side and the drop-aware maintenance paths:** #128, which merges first.
- **#133's other uncovered slices:** shadow promotion, the orphan `test-*` collections, and any VACUUM beyond this run.

## Avoided Traps

- **`update` with documents and no embeddings.** The client re-embeds and silently replaces every vector.
- **A post-hoc FTS drop.** neomjs/neo#14192 showed it leaves the store unwritable.
- **Re-embedding during the backfill.** A byte-identical reconstruction means the stored vector is already correct.
- **Measuring or migrating the live file directly.**

## Decision Record impact

`aligned-with` adr:0027-autonomous-data-recovery-actuator. This migration is operator-run and adopts ADR 0027 §2.4's snapshot-before-mutate and reversibility. It adds no heal action.

## Related

Parent #133 · split from #128, which blocks this · #1432 (defrag and the store's live shape) · neomjs/neo#14192 (the FTS drop, falsified).

Live latest-open + closed sweep: checked the latest 20 open issues plus closed searches ("turn document backfill", "stored documents existing rows", "chroma document reclaim") at 2026-11-02T23:33Z. No equivalent or prior exclusion found.
MC sweep: "existing memory rows still store duplicated turn text in the Chroma document; store too large; reclaim bytes", 6 results. Only #128's history and the June bloat analysis came back; no prior decision on the mechanism.
Own-assignment sweep: 6 open. #128 (the source) and #133 (the parent) are same-surface; neither already covers this.
A2A claim sweep: the last 30 messages hold no claim on this scope.

Origin Session ID: `0d705b6b-6706-4013-b114-91e7490cf576`

## Handoff Retrieval Hints

- `query_raw_memories(query="turn document de-dup backfill existing rows chroma reclaim")`
- `query_raw_memories(query="FTS index drop unwritable compaction")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-02T23:28:48Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-02T23:28:49Z @neo-opus-ada added the `enhancement` label
- 2026-11-02T23:28:50Z @neo-opus-ada added the `ai` label
- 2026-11-02T23:28:50Z @neo-opus-ada added the `architecture` label
- 2026-11-02T23:28:50Z @neo-opus-ada added the `performance` label
- 2026-11-02T23:28:58Z @neo-opus-ada added parent issue #133
- 2026-11-02T23:29:00Z @neo-opus-ada marked this issue as being blocked by #128
- 2026-11-02T23:30:16Z @neo-opus-ada cross-referenced by #128
- 2026-11-02T23:58:29Z @neo-opus-ada cross-referenced by PR #1447

