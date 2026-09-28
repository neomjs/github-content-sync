---
id: 597
title: Corpus KB retains multiple revisions of the same discussion body
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-28T09:34:15Z'
updatedAt: '2026-09-28T12:00:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/597'
author: neo-gpt-emmy
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
closedAt: '2026-09-28T12:00:38Z'
---
# Corpus KB retains multiple revisions of the same discussion body

## Context

During Fleet Manager's Golden Path/KB freshness investigation on 2026-09-28, one discussion body was found as five separately readable KB rows. This issue is the current work record; its evidence is complete here.

## The Problem

For the exact tuple:

- tenant: `neo-shared`
- ingestion repository: `github-content-sync`
- name: `neo/discussion-19151#body`
- source: `neo/discussions/chunk-3/discussion-19151.md`

a scoped Chroma read returned five distinct IDs and content hashes. Their embedded `updatedAt` values are 2026-09-25 10:07 and 10:19, September 26 09:50 and 18:28, and September 27 13:51 UTC. Two use the previous extraction identity and three the current one. The oldest (`9566c804…`) and newest (`2c4b38a7…`) both return through the ordinary `get_document_by_id` MCP tool.

**Established:** retained revisions remain readable, the source-resolution operation selects an old version, and an A→B regression through the owning ingestion/vector/read methods reproduces the defect. Full semantic ranking is not part of this reproduction.

This is distinct from #417's old `neo`-owned conversation rows and #419's old core-source rows. Discussion 19151 is absent from the frozen Engine content tree. GitHub organization/repository duplication was also falsified for this publisher: the five repository discussion connections return 308 for `neo` and zero for the other four origins.

## First source-read falsifier — 2026-09-28

The collection lookup used by `QueryService.findDocBySource` was replayed against the live collection, restricted to the authorized `neo-shared` scope and the exact source path:

`collection.get({where: {$and: [{tenantId: 'neo-shared'}, {source: 'neo/discussions/chunk-3/discussion-19151.md'}]}, limit: 1, include: ['metadatas']})`

It selects body ID `9566c804865840b80806a008d6394e331f0a505ff283450d379693ba038b66ad`, whose embedded update timestamp is **2026-09-25 10:07:14 UTC**, while the later body `2c4b38a7aea7a63961f840c0b0d4039054fec50f34545cb126b96a6f37d9c82a` (**September 27 13:51:02 UTC**) is also present.

This reproduces stale selection in the source-resolution operation; the full semantic-query pipeline was not exercised by this probe. The path contains 42 rows across body/comment elements and revisions — that is **not** a count of 42 duplicate bodies. The original exact-name body count remains five.

### Owning defect and repair boundary

The trusted profile builder emits the complete chunk set for each changed source. Ingestion passes `deleteStale: false`; vector IDs include the content hash. Neither that additive write nor path-only manifest reconciliation retires older IDs while the source path remains present.

The regression first returned **revision A after a successful revision B ingestion**. The repair enables source replacement only for trusted profile input with no parsing/validation errors. It captures prior profile-stamped IDs within the exact tenant/repository and supplied paths, then retires them only after the current batch completes without failures, poison or a yield. Unchanged current elements, other paths/repositories/tenants and unprofiled legacy rows survive. A clean retry can retire old IDs even when all new vectors landed in an earlier partial attempt.

The isolated Chroma control falsified using `extractionIdentity != ''` as proof of field presence: absent fields also match. The repair reads bounded candidate metadata and validates the actual profile stamp. It does not select currentness by ingestion timestamp.

## The Architectural Reality

- `ConversationCorpusSource.createChunk` preserves origin, logical name, source path and per-element identity.
- `VectorService` owns chunk IDs and stale strategies; `IngestionService` and `helpers/kbReconciliationEngine.mjs` own tenant ingestion/manifests.
- `QueryService` owns reference selection, including a source lookup using `collection.get({where, limit: 1})`; `SearchService` resolves selected reference content.
- `DocumentService.getDocumentById` can read both measured IDs.

The investigation belongs to these existing read/write paths, not a new indexing service.

## The Fix

First reproduce a corpus body change from revision A to B through the existing tenant ingestion path. Trace ID replacement, manifest reconciliation and ordinary retrieval. If superseded content can be selected as current, correct the owning step and add the failing regression. Do not infer that retaining an ID is itself wrong if historical access is intentional: prove how current reads exclude it, or fix that separation.

## Contract Ledger

| Surface | Authority | Required result | Fallback / evidence |
|---|---|---|---|
| Current corpus retrieval | Admitted repository revision and logical element | The result represents the admitted current body | A missing current body is reported, not replaced silently with older content; revision-change control |
| Tenant reconciliation | Existing tenant/repository manifest | Replacement is scoped to the affected source/element | Failed/incomplete ingestion and unrelated-row preservation controls |

## Acceptance Criteria

- [x] AC-1 Record the exact A→B reproduction and which ordinary read returns which revision. Name the responsible write/read step, or provide the falsifier showing current retrieval already excludes historical IDs.
- [ ] AC-2 If a current-read defect is confirmed, its regression fails before the repair and passes afterward; repeat unchanged B to verify idempotence.
- [ ] AC-3 Verify failed/partial ingestion and unrelated repositories/source elements preserve valid data. Any maintenance needed for existing rows has a separate explicit scope and before/after receipt.

## Out of Scope

Legacy retirement (#417/#419), Engine content deletion, GitHub discussion acquisition, Golden Path scheduling, and a wholesale KB rebuild.

## Avoided Traps

Do not deduplicate by bare issue/discussion number, rewrite ownership to the origin repository, or delete a whole tenant to remove old revisions.

Decision Record impact: none proposed; preserve the ownership/origin contract of #402.
Structure map: existing `ai/services/knowledge-base/` read/write/reconciliation owners; no new module placement prescribed.

Related: #402, #417, #419.

Owner: Emmy; self-assigned after Institution #307 was installed and verified. Active repair branch: `codex/597-corpus-revisions`. The live corpus has not been mutated; deployment and any replay of unchanged historical files need their own explicit scope and before/after receipt.

Creation sweep: latest 20 live open issues, corpus/revision and duplicate history, #417/#419 bodies, recent all-state A2A and own assignments checked 2026-09-28; no equivalent. Own Brain assignments #426/#306/#48 concern other surfaces. MC turn `702ef86b-27fa-4691-930f-f860c4160ce7` carries the exact specimen and ruled-out acquisition hypothesis.

Origin Session ID: 23b22a41-52ac-4e6c-8d80-23d54054c48c
Retrieval Hint: `corpus discussion-19151 five body revisions same source`
Authored by Emmy (GPT-6 Astra, Codex).



## Timeline

- 2026-09-28T09:34:16Z @neo-gpt-emmy added the `bug` label
- 2026-09-28T09:34:17Z @neo-gpt-emmy added the `ai` label
- 2026-09-28T09:34:17Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-28T09:34:18Z @neo-gpt-emmy cross-referenced by #17416
- 2026-09-28T09:37:38Z @neo-opus-vega cross-referenced by #442
- 2026-09-28T10:35:55Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-28T11:06:25Z @neo-gpt-emmy cross-referenced by PR #602
- 2026-09-28T12:00:38Z @tobiu referenced in commit `c54fd81` - "Merge pull request #602 from neomjs/codex/597-corpus-revisions

fix(kb): replace superseded profile source revisions (#597)"
- 2026-09-28T12:00:39Z @tobiu closed this issue

