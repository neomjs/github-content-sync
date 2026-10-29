---
id: 1350
title: 'github-content-sync re-ingests all 56,454 records every sweep at an unchanged head'
state: CLOSED
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-10-29T13:47:16Z'
updatedAt: '2026-10-29T18:34:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1350'
author: neo-opus-vega
commentsCount: 2
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
closedAt: '2026-10-29T18:34:28Z'
---
# github-content-sync re-ingests all 56,454 records every sweep at an unchanged head

## Context

A resource sampler reads the live plane's container cgroups every 30 s. Laying the orchestrator log over its 2026-10-29 samples, one tenant-repo sync stands out: `neo-shared/github-content-sync` rebuilds its whole corpus on every due sweep, although its head has not moved. The plane runs Brain `436aec63`, and the code paths below are unchanged on `dev` (`2ed12394`).

## The Problem

On 10-29, five consecutive sweeps logged the same line, at 10:27, 10:58, 11:29, 12:01 and 13:35Z:

`neo-shared/github-content-sync materialized: envelopeFiles=56392 envelopeDeleted=0 ingested=56454 deleted=0 embeddings=0 errors=0`, then `completed: head=f414d87c ingested=56454 deleted=0` (~58 s each).

The four other tenant repos log `ingested=0` at their unchanged heads. The 13:34:12–13:35:13Z sweep lines up with a 588 MB receive tick from Chroma (13:34:50) and a 528 MB orchestrator write tick (13:35:20). At one sweep per ~31 min, that is about 27 GB/day of Chroma reads and 24 GB/day of disk writes, for zero changed records.

## The Architectural Reality

*Corrected 2026-10-29 ~16:35Z: the identity hypothesis below was falsified by reading the code, before anything was implemented.*

- **The persisted checkpoint is current.** `tenant-repo-sync-revisions.json` holds `lastIngestedRev: f414d87c…` (the head), a committed materialization attempt, `corpusOutstanding.state: complete` with `remaining: 0`, and `consecutiveFailures: 0`.
- **It is not an extraction-identity mismatch.** `buildIngestEnvelope` throws `KB_INGEST_ENVELOPE_EXTRACTION_IDENTITY_MISMATCH` when the repo's identity differs from the derived one (`tenantRepoIngestEnvelopeBuilder.mjs:432`), and the commit stores that same derived identity. A completed sweep therefore cannot leave the two unequal. The base revision is the persisted `lastIngestedRev`, not null.
- **The cause is delta safety.** The corpus extractor `ConversationCorpusSource` is `deltaSafe: false` (`ExtractorCatalogue.mjs`). On every linear advance the builder sets `fullMaterialization = !profileDeltaSafe` (`tenantRepoIngestEnvelopeBuilder.mjs`, the linear branch), and that includes an advance that changed nothing (base === head). The other four tenant repos use delta-safe extractors, which is why they log `ingested=0`.
- **The runner already handles the empty case.** `selectAssignments` gives an extractor that is not delta-safe all its assignments when anything changed and none when nothing did (`extractionProfileRunner.mjs:401–407`). The builder's blanket full materialization pre-empts that rule.

## The Fix

In the builder's linear branch, a profile that is not delta-safe re-materializes in full only when the advance changed something:

`fullMaterialization = !profileDeltaSafe && (diff.addedOrChanged.length > 0 || diff.deleted.length > 0)`

An empty advance becomes a zero-delta incremental envelope (`changedPaths: []`, `deleted: []`, no manifest), the path that delta-safe tenants already take on every sweep. Identity changes and incomplete checkpoints still reach the builder as a null base and replay in full. Extraction is a function of the tree at the head plus the extraction identity, and the hierarchy identity is part of that identity, so an unchanged commit at an unchanged identity has nothing to re-materialize.

Decision Record impact: none.

## Acceptance Criteria

- [x] AC-1: The cause of the 10-29 replays is named on this ticket: non-delta-safe full materialization at an unmoved head, not an identity mismatch.
- [ ] AC-2: A unit spec shows that a non-delta-safe profile with base === head builds a zero-delta incremental envelope (no `manifestSnapshot`, empty `changedPaths` and `deleted`), and that the same profile with a non-empty diff still builds a full materialization.

## Post-Merge Validation

Moved 2026-10-29 to neo-agent-brain#64 AC-17: the unchanged-head sweep reading and the combined 24-h traffic reading both need a plane running the merged head, and #64 is the receipt ledger that survives this ticket's close.

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
### @neo-opus-vega - 2026-10-29T16:24:33Z

**More cost evidence (2026-10-29 ~16:23Z):** a scoped one-shot of the existing tenant-sync CLI (`--repo-slug github-content-sync`, default Node heap) exited 134 with V8's "Reached heap limit" at about 1.94 GB used of about 2.11 GB, right after `Refreshing neo-shared/github-content-sync`. It wrote no completion receipt.

The replay from a null base that this ticket describes therefore needs more than the default heap just to re-ingest an unchanged corpus. Only the orchestrator's larger process budget completes it on schedule; it peaked at 5.5 GiB heap in the plane sampler's 38.5 h window. An incremental sweep at an unchanged head should need neither.

— Vega (Claude Opus 5.5, Claude Code) 🌿

### @neo-opus-vega - 2026-10-29T16:31:19Z

**Session handover (2026-10-29 ~16:40Z, solo-refresh).** Still mine; nothing is implemented yet. The body states the corrected cause and the fix.

Pickup:
1. Branch from Brain `dev`. In `ai/services/knowledge-base/helpers/tenantRepoIngestEnvelopeBuilder.mjs`, linear branch: `fullMaterialization = !profileDeltaSafe && (diff.addedOrChanged.length > 0 || diff.deleted.length > 0)`. Keep the existing comment and extend it by one clause: an empty advance has nothing for non-delta-safe extractors to depend on.
2. AC-2 spec beside the builder's existing specs. Cover two cases:
   - base === head with a `deltaSafe:false` profile gives an incremental envelope with no `manifestSnapshot` and empty `changedPaths` and `deleted`;
   - a non-empty diff gives a full materialization.

   The runner's `selectAssignments` (`extractionProfileRunner.mjs:401–407`) already returns `[]` for a non-delta-safe extractor on an empty change set.
3. Cross-family review. It touches no `ai/` config, so ADR 0019 does not gate it.

The post-merge reading pairs with the merged #1352: the arm R sampler keeps running to Nov 3.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-10-29T17:40:47Z @neo-opus-vega cross-referenced by PR #1362
- 2026-10-29T17:41:05Z @neo-opus-vega cross-referenced by #64
- 2026-10-29T18:34:28Z @tobiu referenced in commit `c2d061b` - "fix(kb): a non-delta-safe profile stays incremental when no file changed (#1350) (#1362)

ConversationCorpusSource is deltaSafe:false, so the envelope builder turned
every linear advance into a full materialization, including an advance that
changed nothing: github-content-sync re-ingested all 56,454 corpus records
every ~31 min at an unchanged head. A non-delta-safe profile now
re-materializes in full only when the diff is non-empty. An empty advance
builds the zero-delta incremental envelope that delta-safe tenants already
build, and the runner hands a non-delta-safe extractor no assignments for it.
Identity changes and incomplete checkpoints still reach the builder with a
null base and replay in full.

The spec's raw-text parser profile becomes a helper shared by the full-replay
test and the new zero-delta test. Two comments in the spec drop their ticket
refs because the archaeology guard reads the whole file."
- 2026-10-29T18:34:29Z @tobiu closed this issue

