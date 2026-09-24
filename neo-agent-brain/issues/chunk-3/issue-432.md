---
id: 432
title: 'The tenant envelope spawns one git show per file, so each partial slice spends ~80 s re-reading the corpus'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T12:52:41Z'
updatedAt: '2026-09-24T14:24:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/432'
author: neo-opus-vega
commentsCount: 0
parentIssue: 64
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-24T14:24:02Z'
---
# The tenant envelope spawns one git show per file, so each partial slice spends ~80 s re-reading the corpus

## Context

This is lever 2 of #430, split out so that each lever ships with its own seam and witness (child of #64). #430 keeps lever 1: a clean partial repo is due at the next sweep instead of after the global cadence. This leaf owns the cost of each of those slices.

## The Problem

Measured on `neo-local-canonical` (`b99ea11`) for the `github-content-sync` tenant on 2026-09-23:

- Slice 1 (11:48:45–11:54:01Z) and slice 2 (12:20:01–12:25:10Z) both logged `materialized: envelopeFiles=47140 envelopeDeleted=0 ingested=47182 … errors=0`, then `partial-progress: slice budget reached, checkpoint held at none`. They landed 120 and 140 embeddings.
- `lastIngestedRev` stays `null` until the corpus is exhausted, so `buildIngestEnvelope` takes the full path on every slice: it refreshes the mirror and reads all ~47k files before the first embedding batch runs.
- The KB log splits a later slice (13:56:54–14:01:59Z) three ways:
  - the refresh and envelope build take 1 min 46 s;
  - loading the 47,378 chunks and diffing them against the 260 already embedded takes 4 s;
  - embedding takes 3 min 15 s (about 0.7 chunks/s) until the cooperative yield.

**Most of the rebuild is process spawns, not the corpus.** `readRevisionBlob` runs one `git show <rev>:<path>` process per file (`gitMirror.mjs:1454`), and the envelope reads every file through it. On 2026-09-24 I measured the warm projection mirror of the same corpus (1,000 facets, 23.5 MB):

| Read path | 1,000 files | Per file |
|---|---|---|
| one `git show` per file | 1,738 ms | 1.74 ms |
| one `git cat-file --batch` | 107 ms | 0.11 ms |

At 47,140 files that is about 82 s against about 5 s. So the reads make up most of the 1 min 46 s that each slice repeats.

## The Architectural Reality

- The checkpoint contract is all-or-nothing on `lastIngestedRev` (`tenantRepoCheckpointValidity.mjs`). Advancing it on a partial slice would claim the corpus is whole when it is not. `partial-progress` exists to prevent that (`TenantRepoSyncService.mjs:2882`).
- Ingestion is idempotent on content-hashed chunk ids, and already-embedded chunks are skipped, so resumption is correct today. The repeated cost is reading the corpus again.
- The envelope reads through `createRepositoryRevisionReader` (`repositoryRevisionReader.mjs:90`), which requires `listRevisionEntries` and `readRevisionBlob`. The bulk `prefetchRevisionBlobs` tier makes the blobs local first, so the per-file reads that follow are local and the spawn is the whole cost.
- The conversation corpus extractor awaits each read inside its loop (`ConversationCorpusSource.mjs:166`), so the spawns run one after another. The measurement above used bare sequential spawns, and production's `runGit` wraps each read in credential and known-hosts setup, so ~82 s is a lower bound.

## The Fix

Read the files of one materialization through one batch process. GitMirror gets a batched read: a single `git cat-file --batch` session per materialization, fed `<rev>:<path>` requests, whose `<oid> <type> <size>` answers carry the bytes. A `missing` answer falls back to today's per-file `readRevisionBlob`, which carries the `credentialRef` for the promisor fetch. The batch session runs without lazy fetch, so credential handling stays in one place. The revision reader holds the session for the lifetime of the materialization and closes it on every exit path.

The reader's `getEntry` also stops scanning the whole entry list for every read. A linear `find` over 47,140 entries is quadratic across the envelope: 3,784 ms in a measured run, against 6.3 ms for a path index built once.

This adds no persisted state and no checkpoint field, and every full materialization gains from it, including the first slice. The earlier fix was to reuse the previous slice's materialization when the head is unchanged. That design is re-measured after this lands. It needs persisted state to skip what is left, so it has to earn that against the measured remainder.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `GitMirror.openRevisionBlobSession` | `gitMirror.mjs` | one anonymous `cat-file --batch` session per materialization, with lazy fetch off; `read` resolves the bytes or `null`, and `close` never rejects | a `null` answer takes the per-file `readRevisionBlob` path, as today. `null` means a missing object, a non-blob, a body over `maxOutputBytes`, a path containing CR, LF or NUL, or a process that has ended | JSDoc on `openRevisionBlobSession` (new); no `learn/` page describes the read path | `gitMirror.spec`: byte parity with the per-file read; `null` answers leave the stream aligned; the discard past the limit; no lazy fetch on a blobless mirror |
| `createRepositoryRevisionReader` | `repositoryRevisionReader.mjs` | one session shared by the reader and its scopes; `close()` ends it; `getEntry` reads a path index | an adapter without `openRevisionBlobSession`, a failed open, or any read after `close` goes per file | JSDoc on the factory (read order and the owner's close duty), `close()`, `createSharedBlobSession` and `getEntry` | `repositoryRevisionReader.spec`: one open across reader and scope, and a per-file read only where the session answers `null` |
| `buildIngestEnvelope` | `tenantRepoIngestEnvelopeBuilder.mjs` | closes the reader after success and after failure | — | none: an internal lifetime; the reader factory's JSDoc states the owner's duty | builder spec: `close` recorded on both paths; the #16045 isolated-HOME arm sees the session's HOME removed |
| envelope `files` | `buildIngestEnvelope` | byte-identical to today | — | none: the contract is unchanged | builder spec: the byte-identical arm (binary blob, spaced path) |

Wire note: additive only. No config leaf, no schema version, no migration, no checkpoint field.

## Acceptance Criteria

- [ ] **AC-1** An envelope whose blobs are local reads them all through one batch session, with no per-file `git show`; the builder spec records which path served each read. A `missing` answer falls back per file, and a mutation control that removes the fallback turns the missing-blob arm red.
- [ ] **AC-2** On a fixture repository, the envelope `files` from the batched path are byte-identical to the per-file path, including a binary blob and a path with spaces. The session is closed after a successful and a failed materialization.
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* On the corpus tenant, the `materialized:` slice's refresh and envelope build drop from ~1 min 46 s. The before and after log timestamps are recorded on #64 AC-6 item 2. Residual-Owner: #64.

## Out of Scope

- Re-admission cadence (#430, lever 1).
- Reusing the previous slice's materialization. It is re-measured after this lands and filed only if the remainder justifies persisted state.
- Embedding throughput, the blobless mirror's first-materialization round trips (#65), and over-ceiling chunks (neomjs/neo#16972).
- The core corpus projection's per-file loop (`coreCorpusProjection.mjs`), which can adopt the same primitive in its own leaf.

## Avoided Traps

- **Advancing the checkpoint on a partial slice.** It claims a remainder landed, and the contract forbids that for a reason.
- **Caching the envelope before measuring the read.** A cache would have saved the whole rebuild by storing state, when about 80 % of the rebuild was process startup.
- **Reading the KB collection count as progress.** `corpusOutstanding` and the `embeddings=` log field are the instruments.

## Related

#64 (parent) · #430 (lever 1, sibling) · #411 / PR #424 · #449 / PR #450 (the bulk prefetch tier) · #65 · `learn/agentos/cloud-deployment/TenantIngestionModel.md`

Live latest-open sweep: the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T12:29Z, plus the two filed since (#429, #430); none is equivalent, and #430 is the sibling this was split from. A2A in-flight sweep (the 30 most recent, 12:29Z–12:44Z): no claim on the envelope path. MC sweep: `query_raw_memories` (6 results, 12:3xZ) returned prior partial-progress rate measurements (2026-08-18) and the over-ceiling-chunk cause (2026-08-19), and no prior decision on materialization reuse. Own-assignment sweep: #417, #23, #64, #65, #429, #430; none is equivalent. Premise re-measured 2026-09-24T13:20Z (the read-path table above).

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("tenant repo sync envelope rebuild git show per file cat-file batch GitMirror readRevisionBlob")`

Authored by Vega (Fable 5.1, Claude Code) 🌿 · re-measured by Vega (Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-23T12:52:41Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T12:52:43Z @neo-opus-vega added the `bug` label
- 2026-09-23T12:52:43Z @neo-opus-vega added the `ai` label
- 2026-09-23T12:52:43Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T12:53:10Z @neo-opus-vega added parent issue #64
- 2026-09-23T12:53:28Z @neo-opus-vega cross-referenced by #430
- 2026-09-23T12:57:22Z @neo-opus-vega cross-referenced by PR #433
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434
- 2026-09-23T14:17:15Z @neo-opus-vega cross-referenced by #438
- 2026-09-23T14:39:40Z @neo-opus-vega cross-referenced by #440
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444
- 2026-09-24T13:35:30Z @neo-opus-vega changed title from **A clean partial slice re-materializes the whole tenant envelope and re-upserts every chunk row before its first embedding batch** to **The tenant envelope spawns one git show per file, so each partial slice spends ~80 s re-reading the corpus**
- 2026-09-24T14:01:18Z @neo-opus-vega cross-referenced by PR #458
- 2026-09-24T14:24:02Z @tobiu referenced in commit `d26aa89` - "Merge pull request #458 from neomjs/vega/432-batched-revision-reads

feat(kb): the tenant envelope reads a revision through one git cat-file session (#432)"
- 2026-09-24T14:24:02Z @tobiu closed this issue

