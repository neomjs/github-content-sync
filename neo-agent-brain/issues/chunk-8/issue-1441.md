---
id: 1441
title: Drop stored documents from existing turn rows and measure the reclaim
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-02T23:28:48Z'
updatedAt: '2026-11-03T14:24:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1441'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-11-03T14:24:51Z'
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

## Intake (Ada, 2026-11-03)

- **Drift probe fired.** 17 files on this ticket's surface changed after its creation. They came from #1447 (#128, this ticket's prerequisite, now closed) and from #1443 and #1449 (A2A messages embedded through the existing drain). None supersedes the prescription.
- **Classification:** `valid-as-written`.
- **Prescription checked:** `ai/scripts/migrations/` (sibling `backfillChromaSharedUserId.mjs`) owns the concern. The only other layer, `ai/scripts/maintenance/defragChromaDB.mjs`, rebuilds from a full physical copy. #1432 measured that a 4.47 GB copy does not fit beside the store on the guest disk.
- **Epic review on #133:** [6591079092](https://github.com/neomjs/neo-agent-brain/issues/133#issuecomment-6591079092).
- **Tech-debt radar: clean.** The ticket backlog returns only #14192's falsified FTS drop, already listed under Avoided Traps. Memory Core holds no abandoned attempt.
- **What AC-1's measurement changes** ([6596490206](https://github.com/neomjs/neo-agent-brain/issues/1441#issuecomment-6596490206)):
  - The mechanism is the in-place update. B loses a whole batch on a crash between its delete and its add.
  - A request killed after it commits still fails on the client. So after any request error, the migration re-reads that batch and classifies it before going on; it never trusts the error.
  - Only VACUUM returns bytes, and VACUUM needs temporary space up to the post-migration file size. AC-4's run plans that headroom with #1432.

## Contract Ledger

| Surface | Consumer | Contract | Verification |
|---|---|---|---|
| `node ai/scripts/migrations/dropTurnDocuments.mjs` (dry-run by default) | the operator, before AC-4 | Scans the memory collection and writes nothing. It reports the counts of turn rows that qualify, those whose stored document differs from `composeTurnDocumentText(metadata)`, those already empty, and non-turn rows, plus the document bytes that would be freed. | unit spec on a scratch collection |
| `--apply` | the operator, AC-4 | Batch by batch, it first appends each qualifying row's `{id, document}` to the snapshot file and flushes it to disk. Then it updates with `''` documents and the row's own vectors. Re-running skips rows that are already empty. | unit spec: vectors byte-identical, 0 embedding calls, idempotent rerun |
| `--restore <snapshot>` | the operator, on regression | Writes each snapshotted document back, with the row's current vector. | unit spec: round trip restores every document |
| The memory collection's rows | every Memory Core reader | Turn rows read back `''` and are rebuilt by `resolveTurnDocumentForRead`. Non-turn rows and mismatched turn rows are untouched. | unit spec plus AC-5 |

## Acceptance Criteria

- [x] The removal mechanism is chosen on a scratch store, and the three deciding measurements are recorded here. A, the in-place update: [6596490206](https://github.com/neomjs/neo-agent-brain/issues/1441#issuecomment-6596490206).
- [ ] The migration drops a turn row's stored document only where `composeTurnDocumentText(metadata)` reproduces it byte for byte. It leaves every other row untouched and keeps each turn's vector byte-identical.
- [ ] It snapshots each row's prior document before mutating it, restores from that snapshot, and is idempotent and resumable.

## Post-Merge Validation

Both need the operator's run against the plane. After this ticket closes, the residual owner is #133.

- [ ] `[L4-deferred — operator handoff needed]` The operator runs `dropTurnDocuments.mjs` against the plane, in three steps:
  1. A dry-run.
  2. `--apply --snapshot <file>`, with the snapshot on a disk with room for about the dry-run's `documentBytes`.
  3. A VACUUM planned with #1432, since it needs temporary space up to the post-migration file size.

  The reclaimed bytes of `chroma.sqlite3` and the full-text index are measured and recorded on #133.
- [ ] `[L4-deferred — operator handoff needed]` Memory search, session summarization and turn display show no regression on migrated rows.

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
### @neo-opus-ada - 2026-11-03T05:48:53Z

**AC-1: the removal mechanism is A, the in-place update.**

Setup: a throwaway Chroma, server 1.0.0 via chromadb 3.5.0's `chroma run` (darwin-arm64 bindings 1.3.4). It held 2,000 seeded turn rows whose documents are in `composeTurnDocumentText` form, embedded by an 8-dimension deterministic function that counts its calls. Writes went in batches of 250, each mechanism on its own copy of the seed, and sqlite was read and VACUUMed on copies only.

| Measurement | A: `update({ids, documents: '', embeddings: preserved})` | B: `delete`, then `add({ids, embeddings: preserved, metadatas})` |
|---|---|---|
| Vectors byte-identical | 2,000 / 2,000 | 2,000 / 2,000 |
| Embedding calls during the run | 0 | 0 |
| Split metadata intact | 2,000 / 2,000 | 2,000 / 2,000 |
| `$contains` on each row's unique token (50 rows) | 50 → 0 | 50 → 0 |
| `chroma:document` metadata rows | 2,000 kept, all `''` | 0 |
| `embedding_fulltext_search` rows | 2,000 kept, empty | 0 |
| Document read-back | `''` | `null` |
| `chroma.sqlite3` after VACUUM (seed: 6.83 MB) | 4.79 MB | 4.60 MB |
| Rows absent after a SIGKILL | **never** | **250, in 2 of 3 runs** |

**Crash runs** (SIGKILL, restart, then every id classified as old, new or missing):
- A, killed between batches (after batch 3): 1,250 old, 750 new.
- A, killed 13 and 14 ms into batch 4: 1,000 / 1,000, because the 250-row update finished before the kill.
- A, one 2,000-row update killed at 5, 15, 30, 50 and 80 ms: all old, all old, all new, all new, all new. Never mixed, never missing.
- B, killed after batch 3: 1,250 / 750 / 0 missing.
- B, killed 26 ms into batch 4's delete: 250 missing.
- B, killed after batch 4's delete returned, before its add: 250 missing.

**Why A:** it is the only mechanism that never leaves a row absent. B loses the whole batch on any crash between its delete and its add. A's cost is one empty `chroma:document` row and one empty full-text row per turn, about 98 bytes a row here (4.79 vs 4.60 MB over 2,000 rows), against the text it frees. Both read-backs are rebuilt on read: `resolveTurnDocumentForRead` returns the stored document only `if (stored)`, so `''` and `null` both fall through to `composeTurnDocumentText` (checked at `origin/dev`).

**Two findings that shape the migration (AC-2, AC-3):**
1. A request killed after it commits still fails on the client ("Failed to connect"): at 30–80 ms the update had been applied. The migration must therefore classify rows by reading them back after any error, never by the request's result.
2. Only VACUUM returns the bytes. Unvacuumed, `chroma.sqlite3` stayed at 8.22 MB under both mechanisms.

Side finding: `chroma run` logs "Received SIGTERM, shutting down service" and its process never exits. Anything that waits on that exit hangs, so it needs a SIGKILL after a grace period.

Next: the migration in `ai/scripts/migrations/` (AC-2, AC-3).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-03T06:04:46Z @neo-opus-ada cross-referenced by PR #1457
- 2026-11-03T07:42:07Z @neo-opus-ada referenced in commit `defd869` - "fix(memory-core): restore turn snapshots through the shared JSONL reader (#1441)

JSON.stringify leaves U+2028 and U+2029 raw, and readline breaks a line
at each, so --restore split the record of a turn holding either character
into two unparseable halves. The restore now reads through
openJsonlStream, which the JSONL reader census requires of every reader
under ai/. The new spec test failed on the previous head with
"Unterminated string in JSON" and passes on this one."
- 2026-11-03T14:24:51Z @tobiu referenced in commit `af7321e` - "feat(memory-core): drop the stored text of existing turn rows (#1441) (#1457)

* feat(memory-core): drop the stored text of existing turn rows (#1441)

Turn rows written before turns stopped storing a document hold their
text twice: as the split metadata fields, and as the Chroma document that
also feeds the full-text index. dropTurnDocuments.mjs clears that
document where composeTurnDocumentText(metadata) rebuilds it byte for
byte, and keeps every row's vector. resolveTurnDocumentForRead rebuilds
the text on read.

The mechanism was measured on a scratch store (#1441, comment
6596490206). An update with '' and the preserved vector never left a row
absent across eight kill points; delete + add lost 250 rows when killed
between the two. A request killed after it commits still errors on the
client, so a failed batch is read back before the run goes on.

Dry-run by default. --apply snapshots each batch to a JSONL file, flushed
to disk, before clearing it; --restore writes a snapshot back; a rerun
skips rows already empty. VACUUM stays the operator's separate step.

* fix(memory-core): restore turn snapshots through the shared JSONL reader (#1441)

JSON.stringify leaves U+2028 and U+2029 raw, and readline breaks a line
at each, so --restore split the record of a turn holding either character
into two unparseable halves. The restore now reads through
openJsonlStream, which the JSONL reader census requires of every reader
under ai/. The new spec test failed on the previous head with
"Unterminated string in JSON" and passes on this one."
- 2026-11-03T14:24:53Z @tobiu closed this issue

