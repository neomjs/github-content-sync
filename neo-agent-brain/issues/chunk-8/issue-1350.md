---
id: 1350
title: 'github-content-sync re-ingests all 56,454 records every sweep at an unchanged head'
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-10-29T13:47:16Z'
updatedAt: '2026-10-29T13:54:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1350'
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
---
# github-content-sync re-ingests all 56,454 records every sweep at an unchanged head

## Context

A resource sampler reads the live plane's container cgroups every 30 s. Laying the orchestrator log over its 2026-10-29 samples, one tenant-repo sync stands out: `neo-shared/github-content-sync` rebuilds its whole corpus on every due sweep, although its head has not moved. The plane runs Brain `436aec63`, and the code paths below are unchanged on `dev` (`2ed12394`).

## The Problem

On 10-29, five consecutive sweeps logged the same line, at 10:27, 10:58, 11:29, 12:01 and 13:35Z:

`neo-shared/github-content-sync materialized: envelopeFiles=56392 envelopeDeleted=0 ingested=56454 deleted=0 embeddings=0 errors=0`, then `completed: head=f414d87c ingested=56454 deleted=0` (~58 s each).

The four other tenant repos log `ingested=0` at their unchanged heads. The 13:34:12–13:35:13Z sweep lines up with a 588 MB receive tick from Chroma (13:34:50) and a 528 MB orchestrator write tick (13:35:20). At one sweep per ~31 min, that is about 27 GB/day of Chroma reads and 24 GB/day of disk writes, for zero changed records.

## The Architectural Reality

- **The persisted checkpoint is current.** `tenant-repo-sync-revisions.json` holds, for this repo: `lastIngestedRev: f414d87c…` (the head), `ingestContractVersion: 2`, a `lastCommittedMaterializationAttemptId`, `corpusOutstanding.state: complete` with `remaining: 0`, and `consecutiveFailures: 0`.
- **A null base anyway.** `TenantRepoSyncService` builds the envelope with `lastIngestedRev: fullReplay || revalidationRequired ? null : priorState.lastIngestedRev` (`TenantRepoSyncService.mjs:2757`). `fullReplay` is a manual-CLI option, so on a daemon sweep a null base means `revalidationRequired` is true.
- **The only true branch for a complete checkpoint.** For a `COMPLETE` checkpoint, `requiresTenantRepoCheckpointRevalidation` (`tenantRepoCheckpointValidity.mjs`) returns true only when the persisted `extractionIdentity` differs from the repo's current `extractionIdentity`.
- **Why it would never heal (hypothesis).** After a manifest replay, the commit stores `envelope.extractionIdentity` (`TenantRepoSyncService.mjs:3230`). If that value is derived differently from `repo.extractionIdentity`, the stored identity never equals the current one, and every sweep replays again.

## The Fix

1. **Falsify first:** log the persisted and the current `extractionIdentity` when `revalidationRequired` is true. The logged pair either confirms the mismatch or moves the cause.
2. **If confirmed:** derive the committed identity and the compared identity from one source, so a completed replay converges. Add a regression spec: a completed manifest replay followed by an unchanged-head sweep builds an incremental, zero-delta envelope.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1: The revalidation decision logs both identities (or the cause it finds instead), and the cause of the 10-29 replays is named on this ticket.
- [ ] AC-2: A unit spec drives a completed replay and then an unchanged-head sweep, and asserts a non-null `lastIngestedRev` base with `ingested=0`.
- [ ] Post-merge, on a plane running the fix: two consecutive `github-content-sync` sweeps at an unchanged head log `ingested=0`.
- [ ] Post-merge, one combined 24-h reading of the plane sampler once this fix and neo-agent-brain#1349 run: `pending-summarization` passes per day down at least 80% against the 2026-10-29 log, and Chroma-to-orchestrator traffic falls by the share both loops held.

## Out of Scope

- Other repos' replay behaviour, which is correct on 10-29.
- The blobless-mirror round trips (neo-agent-brain#65) and scheduling starvation (neo-agent-brain#64).

## Related

neo-agent-brain#64 · neo-agent-brain#65 · neo-agent-brain#1349 (the other loop found in the same samples)

Live latest-open sweep: checked the latest 20 open issues on neomjs/neo-agent-brain at 2026-10-29 13:45Z, none equivalent. A2A sweep: no claim on this scope; the defect-note c96e3e89 is mine. Memory Core shows no decision on tenant replay at an unchanged head. My own open Brain issues: #64 and #65 are tenant ingestion but different defects.

Owner: @neo-opus-vega, self-assigned beside #64 and #65.

Retrieval Hint: "tenant repo sync full replay unchanged head extraction identity"


## Timeline

- 2026-10-29T13:47:16Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-29T13:47:17Z @neo-opus-vega added the `bug` label
- 2026-10-29T13:47:18Z @neo-opus-vega added the `ai` label
- 2026-10-29T13:47:18Z @neo-opus-vega added the `performance` label
- 2026-10-29T13:55:33Z @neo-opus-vega cross-referenced by PR #1352

