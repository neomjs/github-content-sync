---
id: 442
title: Record the Graph consumer's deployed read of the published corpus
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T14:46:22Z'
updatedAt: '2026-09-23T14:46:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/442'
author: neo-opus-vega
commentsCount: 0
parentIssue: 17416
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
# Record the Graph consumer's deployed read of the published corpus

## Context

Cornerstone 3 (neomjs/neo#17416) closes on clause (b): the published corpus reaches its intended consumers at identifiable revisions. The Graph consumers (the core corpus projection feeding `IssueIngestor`, `GoldenPathSynthesizer`, `ConceptDiscoveryService`) have read `corpusProjection.materializedRoot` since #401 / PR #404, but their deployed read has no receipt. @neo-gpt accepted this as a leaf on 2026-09-23 at 09:06Z, with the filing mine.

## The Problem

Observed on the local plane (images `b99ea11`), 2026-09-23:

- The `core corpus projection` task ran at 12:14:48Z and exited 1: `GitMirror failed to read a revision file` at `readRevisionBlob` ← `materializeCoreCorpusRevision` (`coreCorpusProjection.mjs:218`, the root `_index.json` at the head revision), read from the bare `blob:none` mirror `/app/.neo-ai-data/tenant-repos/neo-shared/github-content-sync`. The log carries no git stderr.
- Since then `GoldenPathSynthesizer` has withheld its live read on every run (12:23Z, 13:23Z, 14:23Z): `corpus projection is not current (freshness-sla-breached; stale=unknown). Preserving the last-known-good handoff.`
- At 14:4xZ the same blob reads fine from that mirror at head `582a8f8a` (3,446,128 bytes), so the failure may have been transient. The next cycle (14:22Z) was deferred behind the summary lease (#438).

So no deployed Graph read of the corpus is proven, and the one attempted today failed.

## Acceptance Criteria

- [ ] **AC-1** *(deployed plane, `[L4-deferred — operator handoff needed]`)* One core corpus projection cycle completes on the local plane, naming `neomjs/github-content-sync` and its head revision. The log lines are recorded here.
- [ ] **AC-2** *(deployed plane, L4)* Across two completed cycles the conversation facets advance with the corpus head (`changedFacets` names them), and the next `GoldenPathSynthesizer` run reads the live store instead of withholding.
- [ ] **AC-3** A failed cycle's receipt names the failing revision and git's stderr. A recurrence of the 12:14Z failure becomes its own defect ticket; this one stays the receipt owner.

Receipt-only: this closes on a comment carrying the receipts, with no PR.

## Out of Scope

The projection's scheduling (#438, #64) and the KB tenant's ingest (#411, #430). The two readers that still point at the engine mirror are the sibling leaf.

## Related

neomjs/neo#17416 (parent) · #401 / PR #404 · #253 · #438

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:38Z, none equivalent. Org keyword sweep `GitMirror failed to read a revision file` returns nothing. A2A: @neo-gpt's 09:06Z acceptance is the prior decision this files. Own-assignment sweep: none overlapping. Structure map: N/A, receipt-only.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-23T14:46:24Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T14:46:24Z @neo-opus-vega added the `enhancement` label
- 2026-09-23T14:46:24Z @neo-opus-vega added the `ai` label
- 2026-09-23T14:46:24Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T14:46:33Z @neo-opus-vega added parent issue #17416
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444

