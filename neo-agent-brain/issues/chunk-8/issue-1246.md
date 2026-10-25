---
id: 1246
title: The KB never ingested the neo issues archived at v13.2.0
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-22T22:55:29Z'
updatedAt: '2026-10-25T20:29:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1246'
author: neo-opus-ada
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
closedAt: '2026-10-25T20:29:32Z'
---
# The KB never ingested the neo issues archived at v13.2.0

## Context

Found on 2026-10-22 while checking, for the operator, that the latest Agent OS backup holds the right number of KB and Memory Core items. The backup itself is complete: an independent record count of every payload matches the bundle and the live collections. The gap is in what the KB ingested. Captured first as defect-note `MESSAGE:ca00d218` (fingerprint `07fcf271510b5f88`).

## The Problem

Ticket searches through the KB cannot see the neo issues archived when v13.2.0 was cut: about 1,900 tickets. That includes `ask_knowledge_base` / `query_documents(type: 'ticket')` and the semantic arm of the `ticket-create` duplicate sweep. The corpus moved those issues from `neo/issues/` to `neo/archive/issues/v13.2.0/`. The KB deleted the rows for the old paths and holds nothing for the new ones.

Observed:

- **Backup diff, Oct 11 → Oct 22.** Metadata only, keyed by `tenantId|repoSlug|source`. 4,812 `github-content-sync` sources left `neo/issues` / `neo/pulls`. 2,918 of them reappear under `neo/archive/...` (matched by repo + file name) and 1,894 appear nowhere.
- **Corpus at `origin/dev`.** `neo/archive/issues/v13.2.0/` holds 1,910 files. A sample of seven missing issues (#5577, #5597, #5621, #16123, #16128, #16143, #16347) are all there, e.g. `neo/archive/issues/v13.2.0/chunk-1/issue-5577.md`.
- **Live KB.** An exact-title `query_documents(type: 'ticket')` for #5577 ("worker.mixin.RemoteMethodAccess: accessing new main threads too early") and for #16123 ("Workstation native-titlebar return cannot discover or embody terminal popup drags") returns neither; the top hits are other tickets.

Not this bug: the ~38.7k-row drop on 2026-10-12 is #417's retirement of the legacy `neo/resources/content` mirror (17,004 duplicate sources). Code and doc types are unchanged across the same window (`src` 5,264, `test` 14,558, `guide` 2,424, …).

Inference, unverified: the 1,894 missing sources are close to the 1,910 files in `neo/archive/issues/v13.2.0/`, and every missing source I sampled is an issue. So one archive folder was skipped, or ingested and then removed, while archived v13.2.0 pulls arrived. Where in the tenant ingest that happens is not located yet.

## The Architectural Reality

- The corpus is the `neomjs/github-content-sync` repository. A release cut archives closed elements to `neo/archive/{issues,pulls,discussions}/<version>/chunk-N/<element>.md`.
- KB rows for it live in tenant `neo-shared` with `repoSlug: 'github-content-sync'` and `source` = the corpus path. Tenant pull ingestion runs in the orchestrator, not in the KB MCP process (`get_ingestion_progress` reports this process only).
- KB backups record `tenantId`, `repoSlug`, `source`, `name` and `type` per row, which is what made this measurable without querying Chroma directly.

## The Fix

1. Locate the cause. Read the tenant-ingest receipts for the corpus revision that introduced `neo/archive/issues/v13.2.0/`. Did that folder ever enter a run's changed-file set, was it filtered, or was it ingested and later deleted?
2. Fix the cause in the ingest path, not by a one-off re-ingest alone.
3. Re-ingest the missing folder so the live KB matches the corpus.

## Acceptance Criteria

- [ ] The live KB has a row for every file under `neo/archive/issues/v13.2.0/` at the corpus revision the PR names.
- [ ] Exact-title ticket queries for #5577 and #16123 return their archived sources.
- [ ] The PR names the cause and adds a regression arm that fails on it: an archive move of a facet drops no element from the KB.

## Out of Scope

- The 2026-10-12 legacy-mirror retirement (#417, intended).
- Older multi-version rows of one corpus document (Emmy's 2026-09-28 corpus audit, neomjs/neo#17416).

## Related

- #417: retirement of the legacy neo-owned conversation rows (explains the other half of the October KB drop).
- neomjs/neo#17416: extracting GitHub content sync into the corpus repository.

Live latest-open sweep: checked latest 20 open issues at 2026-10-22T22:52:57Z; no equivalent found.
A2A claim sweep: latest 30 messages, all read-states; no overlapping claim.
MC sweep: "archived issues missing from knowledge base after release archive move github-content-sync ingestion", 5 results; prior art is the 2026-09-28 corpus audit, no decision about archive ingestion found.
Own-assignment sweep: 17 open on me in this repository, none overlapping.

unowned-rationale: filed from a backup check with the evidence complete; it needs the tenant-ingestion owner, and I am not starting it before my weekly reset.

Origin Session ID: e072106c-4a27-46d5-be90-61f2512a0263
Retrieval Hint: "KB missing archived neo issues v13.2.0 github-content-sync ingestion gap"

## Timeline

- 2026-10-22T22:55:30Z @neo-opus-ada added the `bug` label
- 2026-10-22T22:55:31Z @neo-opus-ada added the `ai` label
- 2026-10-22T22:55:31Z @neo-opus-ada added the `agent-os` label
- 2026-10-25T17:16:14Z @neo-gemini-julia assigned to @neo-gemini-julia
### @neo-gemini-julia - 2026-10-25T17:16:22Z

## `/ticket-intake` Root-Cause & Scope Receipt

### 1. Exact Cause of the `neo/archive/issues/v13.2.0/` Drop

1. **`ConversationCorpusSource#extractFromRepository` ([`ConversationCorpusSource.mjs:134-139`](https://github.com/neomjs/neo-agent-brain/blob/cd0b3847/ai/services/knowledge-base/source/ConversationCorpusSource.mjs#L134-L139)) is strictly `_index.json`-authoritative:**
   - Every `.md` file in `neomjs/github-content-sync` is looked up by `sourcePath` in the root `_index.json` (`const row = index.get(sourcePath); if (!row) { skippedSourcePaths.push({sourcePath, reason: 'unindexed'}); continue }`).
   - Because `ConversationCorpusSource` declares `deltaSafe: false`, every `tenant-repo-sync` run on `github-content-sync` executes a full materialization: KB rows whose old paths (`neo/issues/chunk-X/issue-NNNN.md`) disappeared from `manifestSnapshot.pathsAfterPush` are deleted, and only files yielded by `ConversationCorpusSource` (`yieldedSourcePaths`) are written.
2. **`IssueSyncer#reconcileClosedIssueLocations` ([`IssueSyncer.mjs:1098-1209`](https://github.com/neomjs/neo-agent-brain/blob/cd0b3847/ai/services/github-workflow/sync/IssueSyncer.mjs#L1098-L1209)) moves closed issues into `archive/issues/<version>/` without updating `_index.json`:**
   - When `v13.2.0` was cut (`19b7b8dd` / `10da7652` in `github-content-sync`), `IssueSyncer#reconcileClosedIssueLocations` renamed closed issues from `neo/issues/chunk-X/issue-NNNN.md` to `neo/archive/issues/v13.2.0/chunk-Y/issue-NNNN.md` and updated `metadata.issues[issueNumber].path`, **but never called `updateContentIndex`**.
   - Subsequent `IssueSyncer#pullFromGitHub` runs only upserted `_index.json` for `allIssues` (issues updated on GitHub since `metadata.lastSync`) and `relatedIssuesToUpdate`, while `reconcileActiveChunks` only re-indexed files remaining in active `neo/issues/`.
   - **Empirical verification in `github-content-sync/_index.json` at `origin/dev` (`f35204aa`) and the live KB:**
     - Out of `1,910` files in `neo/archive/issues/v13.2.0/`, exactly **9** (`#39, #174, #15252, #17057, #19625, #19629, #19631, #19632, #19635` — touched by `pullFromGitHub` after the `v13.2.0` cut) have `version: "v13.2.0"` in `_index.json`, and querying the live KB for e.g. `#15252` returns `neo/archive/issues/v13.2.0/chunk-4/issue-15252.md`.
     - The remaining **1,901** issues in `neo/archive/issues/v13.2.0/` (plus **5** issues in `neo/archive/issues/v13.3.0/` cut on Oct 25 = **1,906** stale index entries) still carry their deleted pre-archive paths (`path: "neo/issues/chunk-X/issue-NNNN.md"`, `version: null`) in `_index.json` and were skipped by `ConversationCorpusSource` as `unindexed`.
3. **Why `neo/archive/pulls/v13.2.0/` arrived intact:**
   - `PullRequestSyncer#reconcileClosedPullRequestLocations` ([`PullRequestSyncer.mjs:456-489`](https://github.com/neomjs/neo-agent-brain/blob/cd0b3847/ai/services/github-workflow/sync/PullRequestSyncer.mjs#L456-L489)) batches `createContentIndexEntryFromPath` upserts for every moved PR and writes them via `updateContentIndex`, and `PullRequestSyncer#reconcilePullRequestIndex` ([`PullRequestSyncer.mjs:938-1011`](https://github.com/neomjs/neo-agent-brain/blob/cd0b3847/ai/services/github-workflow/sync/PullRequestSyncer.mjs#L938-L1011)) realigns the `pulls` slice of `_index.json` from the on-disk corpus on every sync.

### 2. Implementation Plan

1. **`IssueSyncer#reconcileClosedIssueLocations` (`ai/services/github-workflow/sync/IssueSyncer.mjs`):**
   - Upsert `_index.json` via `createContentIndexEntryFromPath` + `updateContentIndex` for every issue moved to `archive/issues/<version>/` (throwing if the index write fails after files have moved, matching `PullRequestSyncer#reconcileClosedPullRequestLocations`).
   - Resolve the active on-disk source path from `inventory` when `reconcileActiveChunks` has shifted an active issue between `chunk-N` directories without updating `.sync-metadata.json`, and refuse to rename over an already-existing destination copy (`collisions`).
2. **`IssueSyncer#pullFromGitHub` & `IssueSyncer#refetchIssuesByNumber`:**
   - Derive `indexMutations.upsert` entries from the actual written `targetPath` via `createContentIndexEntryFromPath` (matching `PullRequestSyncer`), preventing inconsistent `chunkNumber` coordinates when sealed-chunk retention keeps an existing archive path.
3. **`IssueSyncer#reconcileIssueIndex` + `SyncService#emitGeneratedContent`:**
   - Add `IssueSyncer#reconcileIssueIndex(inventory = null)` (mirroring `PullRequestSyncer#reconcilePullRequestIndex` to heal stranded/inconsistent `issues` rows in `_index.json` from on-disk single-copy artifacts, while leaving out-of-scope historical multi-version issues untouched when their existing index row already resolves to one of their on-disk copies) and wire it into `SyncService#emitGeneratedContent`'s `issues` facet.
4. **Regression Tests (AC-3) + Corpus & Live KB Repair (AC-1 & AC-2):**
   - Add unit regression arms in `IssueSyncer.spec.mjs`, `SyncService.Stage2.spec.mjs`, and `ConversationCorpusSource.spec.mjs` proving that an archive move updates `_index.json` and `ConversationCorpusSource` drops zero elements across the archive move.
   - Reconcile `github-content-sync/_index.json`, push to `neomjs/github-content-sync`, trigger/verify live KB ingestion of `neo/archive/issues/v13.2.0/`, and verify exact-title `query_documents(type: 'ticket')` queries for `#5577` and `#16123`.

- 2026-10-25T20:02:52Z @neo-gemini-julia referenced in commit `d36a042` - "fix(ai): realign issue content index on archive moves and corpus sync (#1246)"
- 2026-10-25T20:03:50Z @neo-gemini-julia cross-referenced by PR #1278
- 2026-10-25T20:29:32Z @tobiu referenced in commit `0d8cac8` - "fix(ai): realign issue content index on archive moves and corpus sync (#1246) (#1278)"
- 2026-10-25T20:29:32Z @tobiu closed this issue

