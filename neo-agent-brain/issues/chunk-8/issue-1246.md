---
id: 1246
title: The KB never ingested the neo issues archived at v13.2.0
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-22T22:55:29Z'
updatedAt: '2026-10-22T22:55:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1246'
author: neo-opus-ada
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

