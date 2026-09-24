---
id: 449
title: 'The corpus projection never bulk-prefetches, so a cold mirror fails'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T11:46:54Z'
updatedAt: '2026-09-24T12:22:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/449'
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
blocking: []
closedAt: '2026-09-24T12:22:03Z'
---
# The corpus projection never bulk-prefetches, so a cold mirror fails

## Context

On the local plane (images `b99ea11`), the first core corpus projection cycle under `github-content-sync.git` (2026-09-24 11:18Z, right after #442's source flip) failed in its per-file loop (`coreCorpusProjection.mjs:271`) with `GitMirror failed to read a revision file`. The cycle cloned its own mirror fresh, and `cloneIfMissing` creates mirrors with `--filter=blob:none`. At the failure, 16,852 of the 18,849 facet blobs at head `3894176` were absent. Every one of those reads is a lazy promisor fetch, eight in parallel (`readConcurrency`).

Measured from inside the orchestrator container: of 16 such fetches run in parallel, 14 failed with `Failed to connect to github.com port 443 … could not fetch <oid> from promisor remote`, while sequential `ls-remote` calls all succeeded. One `prefetchRevisionBlobs` run over the same paths then fetched the remaining 11,849 blobs in 12 chunks in 9.4 s, after an earlier run stopped at its sixth chunk on the same transient refusal. The next cycle read everything locally.

## The Problem

A cold projection mirror cannot complete its first cycle. That covers a rebuilt plane, a new operator's plane (the first-run path of neomjs/neo#18965) and any change of source repository: each starts a mirror with no blobs, and the projection reads them one network round trip at a time. #247 solved exactly this for the tenant ingest path and scoped itself to that caller. The projection arrived later (#401) and reads through `readRevisionFile` alone.

The failed cycle also left `core-corpus-materialized.next-<pid>-<uuid>` behind, holding one file. `mapWithConcurrency` rejects on the first failure while its sibling workers keep reading and writing. The caller's `finally` removes the staging directory, and a sibling's `outputFile` then recreates it, so every failed full cycle can orphan a staging directory.

## The Architectural Reality

- `materializeCoreCorpusRevision` (`ai/daemons/orchestrator/services/coreCorpusProjection.mjs:199–300`) reads the root `_index.json` (:218). A full cycle then reads every facet path through `gitMirror.readRevisionFile` inside `mapWithConcurrency(addedOrChanged, readConcurrency, …)` (:271); an incremental cycle reads only `diff.addedOrChanged`.
- `GitMirror.prefetchRevisionBlobs` (`ai/services/knowledge-base/helpers/gitMirror.mjs:1278–1404`) is the bulk tier. It does one `ls-tree`, probes for missing blobs with `rev-list --missing=print` without fetching, then runs chunked `git fetch origin <oids…>`. It never rejects: `unavailable` leaves the per-file reads exactly as capable as before. It is scoped to the paths it is given, never the whole tree.
- The projection receives `gitMirror` as a seam (`runCoreCorpusProjectionCycle({gitMirror = GitMirror})`), and its spec already drives it with a fake.

## The Fix

In `materializeCoreCorpusRevision`, once the paths to read are known and before the first blob read, call `gitMirror.prefetchRevisionBlobs` with the root index and those paths. Full and incremental cycles pass their own path lists, so an incremental cycle prefetches only what changed. Carry the prefetch outcome (`status`, `missing`, `chunks`, `reason`) on the materialization result so the child's outcome line reports it. The per-file reads stay as the fallback tier. `mapWithConcurrency` stops taking new values after the first failure and rejects only once its in-flight calls have settled.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| blob acquisition in `materializeCoreCorpusRevision` | #247's two-tier contract in `gitMirror.mjs` | one bulk prefetch of the index and the paths about to be read, before the first read | `unavailable` → per-file lazy reads, as today | unit arm with a fake mirror recording call order |
| incremental cycle | same | prefetch scoped to `diff.addedOrChanged` | — | unit arm asserting the passed paths |
| materialization result | `coreCorpusProjection.mjs` | carries the prefetch outcome | — | unit arm |

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1** A full materialization calls `prefetchRevisionBlobs` once, with the root index and every path it will read, before the first `readRevisionFile`. Unit witness with a fake mirror that records call order, red on the current code.
- [ ] **AC-2** An incremental cycle prefetches only its changed paths, never the whole tree. Unit witness.
- [ ] **AC-3** A prefetch that reports `unavailable` changes nothing about the reads that follow, and the result carries the outcome. Unit witness.
- [ ] **AC-4** A failed read stops the pool from taking new paths and lets in-flight reads settle before the staging directory is removed, so no `.next-*` directory is left behind. Unit witness, red on the current pool.

Evidence ceiling L2. The mechanism was measured on the plane above: the reads succeed once the blobs are local, and the 9.4 s bulk fetch was done by hand. The next cold mirror is the deployed confirmation, and #442 holds the plane receipts.

## Out of Scope

- The projection's scheduling and the lease it holds (#64).
- The tenant ingest path, which already prefetches (#247).
- Retrying a transient refusal inside the prefetch; its contract is to degrade, not retry.

## Avoided Traps

- **Lowering `readConcurrency` to survive the refusals.** Sixteen thousand sequential round trips at ~0.4 s each is close to two hours under the heavy-maintenance lease.
- **Prefetching the whole tree on every cycle.** An incremental cycle would then pay the first-ingest bill every two hours (#247's scoping rule).

## Related

#442 (receipt owner; the source flip) · #247 (the ingest path's prefetch) · #401 (the projection's corpus-repo read) · #65 · #448 · neomjs/neo#18965 · neomjs/neo#17416

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-24T11:44Z; none equivalent. Org exact search (`prefetchRevisionBlobs`, `"promisor remote"`, `"core corpus projection"`) found closed #247 (ingest path only; its body names the envelope builder as "the sole caller") and closed neomjs/neo#16631 / neomjs/neo#16546, none covering the projection. A2A in-flight sweep (12 most recent, to 11:45Z): no overlapping claim. MC sweep: `query_raw_memories` on the symptom (6 results): the 2026-09-21 #401 cutover notes and no prior decision on projection prefetch. Own-assignment sweep: #65 is the tenant-ingest observation, deprioritised and a different caller, and #442 routes here. Structure map: N/A, no new file.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("core corpus projection cold blobless mirror lazy promisor fetch fails prefetchRevisionBlobs")`

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-24T11:46:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-24T11:46:56Z @neo-opus-vega added the `bug` label
- 2026-09-24T11:46:57Z @neo-opus-vega added the `ai` label
- 2026-09-24T11:46:57Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T11:48:06Z @neo-opus-vega cross-referenced by #442
- 2026-09-24T11:54:41Z @neo-opus-vega cross-referenced by PR #450
- 2026-09-24T12:05:15Z @neo-opus-vega cross-referenced by PR #452
- 2026-09-24T12:22:03Z @tobiu referenced in commit `1e5655e` - "Merge pull request #450 from neomjs/vega/449-projection-prefetch

fix(orchestrator): the corpus projection bulk-prefetches its blobs before the first read (#449)"
- 2026-09-24T12:22:03Z @tobiu closed this issue
- 2026-09-24T13:35:31Z @neo-opus-vega cross-referenced by #432

