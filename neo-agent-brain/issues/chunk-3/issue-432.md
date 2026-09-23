---
id: 432
title: A clean partial slice re-materializes the whole tenant envelope and re-upserts every chunk row before its first embedding batch
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T12:52:41Z'
updatedAt: '2026-09-23T12:52:41Z'
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
---
# A clean partial slice re-materializes the whole tenant envelope and re-upserts every chunk row before its first embedding batch

## Context

Lever 2 of #430, split out so each lever ships with its own seam and witness (child of #64). #430 keeps lever 1 — a clean partial repo is due at the next sweep instead of after the global cadence. This leaf owns the cost of each of those slices.

## The Problem

Measured on `neo-local-canonical` (`b99ea11`) for the `github-content-sync` tenant, 2026-09-23: slice 1 (11:48:45–11:54:01Z) and slice 2 (12:20:01–12:25:10Z) both logged `materialized: envelopeFiles=47140 envelopeDeleted=0 ingested=47182 … errors=0` and then `partial-progress: slice budget reached, checkpoint held at none`, landing 120 and 140 embeddings respectively. `lastIngestedRev` stays `null` until the corpus is exhausted, so `buildIngestEnvelope` takes the full path on every slice: 47,140 files materialized from the mirror and 47,182 chunk rows upserted before the first embedding batch runs. Most of the 5-minute `sliceBudgetMs` pays for work whose result is identical to the previous slice's, and the embedding stage — the only stage that advances `corpusOutstanding` — gets the remainder.

With #430's lever 1 the slices run back to back, which makes this cost the dominant term: ~4 minutes of rebuild for ~1 minute of embedding per slice, and the heavy-maintenance lease held for the rebuild too.

## The Architectural Reality

- The checkpoint contract is deliberately all-or-nothing on `lastIngestedRev` (`tenantRepoCheckpointValidity.mjs`): advancing it on a partial slice would claim a corpus is whole when it is not — the failure `partial-progress`'s own comment names (`TenantRepoSyncService.mjs:2865–2975`). So the reuse cannot be "advance the checkpoint early".
- The ingestion path is idempotent on content-hashed chunk ids; already-embedded chunks are skipped. Resumption is correct today — the envelope build and the row upsert are the repeated cost, not the embedding.
- `tenantRepoIngestEnvelopeBuilder.mjs` resolves head and base revisions against the mirror and materializes files under the orchestrator's writable layer; the head revision is known before any file is read.

## The Fix

Persist the materialized head beside the resume marker (`partialHead`, `normalizeTenantRepoCheckpointState` allowlist, validated whole or dropped). When a clean partial repo comes due and the mirror's resolved head equals `partialHead`, skip materialization and the row upsert and hand the embedder the outstanding chunks only; a moved head, an absent marker, or any failure invalidates it and rebuilds as today. The reuse must never mint or reuse a full-materialization receipt — that is the shortcut `partial-progress` returns before `assertFullMaterializationEffect` to prevent.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| persisted `partialHead` | `normalizeTenantRepoCheckpointState` allowlist | survives reload; validated whole or dropped; cleared on `complete`, failure, deferral, stop | `null` — a half-record cannot skip a rebuild | mutation: removing it from the allowlist reddens the second-slice witness |
| envelope build on an unchanged head | `tenantRepoIngestEnvelopeBuilder` / `TenantRepoSyncService` | reuse the materialization, embed outstanding chunks only | head moved or marker absent → full rebuild as today | log receipt names the reuse; `envelopeFiles`/`ingested` work is zero on the reused slice |
| full-materialization receipt | `assertFullMaterializationEffect` | never minted or reused by a reused slice | — | existing AC-6 negative arm stays green |

Wire note: additive only; no config leaf, no schema version, no migration.

## Acceptance Criteria

- [ ] **AC-1** Two consecutive clean partial slices on an unchanged mirror head do not re-materialize the envelope: the second slice's receipt names the reused materialization and its `envelopeFiles`/`ingested` work is zero; a moved head rebuilds. Unit witness plus the mutation control on the checkpoint allowlist.
- [ ] **AC-2** A reused slice never mints or reuses a full-materialization receipt; the checkpoint still advances only when the corpus is exhausted (the existing negative arm stays green, and a reused slice's `lastIngestedRev` is unchanged).
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* with #430 and this leaf deployed, the corpus tenant's `lastIngestedRev` is set and `corpusOutstanding.state: complete` within one day at ~0.6 chunks/s measured throughput; recorded on #64 AC-6 item 2 with the snapshot timestamp. Residual-Owner: #64.

## Out of Scope

- Re-admission cadence (#430, lever 1).
- Embedding throughput, the blobless mirror's first-materialization round trips (#65), over-ceiling chunks (neomjs/neo#16972).

## Avoided Traps

- **Advancing the checkpoint on a partial slice.** It claims a remainder landed; the contract forbids it for a reason.
- **Keying the reuse on elapsed time.** Only the head revision says whether the materialization is still the corpus.
- **Reading the KB collection count as progress.** `corpusOutstanding` and the `embeddings=` log field are the instruments.

## Related

#64 (parent) · #430 (lever 1, sibling) · #411 / PR #424 · #65 · `learn/agentos/cloud-deployment/TenantIngestionModel.md`

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T12:29Z plus the two filed since (#429, #430) — none equivalent; #430 is the sibling this was split from. A2A in-flight sweep (30 most recent, 12:29Z–12:44Z): no claim on the envelope path. MC sweep: `query_raw_memories` (6 results, 12:3xZ) — prior partial-progress rate measurements (2026-08-18) and the over-ceiling-chunk cause (2026-08-19); no prior decision on materialization reuse. Own-assignment sweep: #417, #23, #64, #65, #429, #430 — none equivalent. Structure map (12:3xZ): `ai/services/knowledge-base/helpers/tenantRepoIngestEnvelopeBuilder.mjs` owns the build; `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs` (2,103 code lines) owns the outcome branch — the reuse belongs in the builder, not as new lines on the service.

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("tenant repo sync partial slice re-materializes envelope partialHead reuse materialization corpus tenant")`

Authored by Vega (Fable 5.1, Claude Code) 🌿

## Timeline

- 2026-09-23T12:52:41Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T12:52:43Z @neo-opus-vega added the `bug` label
- 2026-09-23T12:52:43Z @neo-opus-vega added the `ai` label
- 2026-09-23T12:52:43Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T12:53:10Z @neo-opus-vega added parent issue #64
- 2026-09-23T12:53:28Z @neo-opus-vega cross-referenced by #430
- 2026-09-23T12:57:22Z @neo-opus-vega cross-referenced by PR #433
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434

