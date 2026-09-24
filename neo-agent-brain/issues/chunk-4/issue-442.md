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
updatedAt: '2026-09-24T14:06:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/442'
author: neo-opus-vega
commentsCount: 3
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

Observed on the local plane (images `b99ea11`). After the #253 cut, every core corpus projection cycle failed: 3/3 (2026-09-23 12:14Z and 18:41Z, 2026-09-24 10:38Z). Each failed on `GitMirror failed to read a revision file` at `materializeCoreCorpusRevision` (`coreCorpusProjection.mjs:218`), reading the root `_index.json`. The last success was 2026-09-23 03:56Z, on the pre-cut images. Meanwhile `GoldenPathSynthesizer` withheld its live read on every run: `corpus projection is not current (freshness-sla-breached; …). Preserving the last-known-good handoff.` Two causes, found in this order:

1. **The source was still neo.git.** For the window, the #253 cut fragment pinned `NEO_ORCHESTRATOR_CORPUS_SOURCE_REPOSITORY` to `https://github.com/neomjs/neo.git` (#253 comment 5785958556). The cutover receipt says the pin "retires with the #411 activation"; it did not. Since #401 the projection reads the corpus-repo layout, a root `_index.json` plus `neo/`-prefixed facets, and neo.git has neither. In the projection's own mirror (`orchestrator-daemon/core-corpus-mirror`, remote neo.git), `git show 18b473b6e8:_index.json` exits 128 with `fatal: path '_index.json' does not exist`. **Retired 2026-09-24 11:13Z, operator-approved.** Both pins were dropped from the fragment, so the compose default `github-content-sync.git` applies to orchestrator and mc-server. Both were recreated env-only on the same `b99ea11` images, leaving all four Brain containers on one Brain SHA and one neo pin (`17b59aad`). The recreate also dropped the orchestrator's writable-layer state (the #425 class). It was restored at 12:16Z, but writes from 2026-09-23 11:44Z to 2026-09-24 11:13Z are lost: incident comment 5814268631.
2. **A cold blobless mirror cannot be read one file at a time.** The first cycle under the new source (11:18Z) cloned `github-content-sync` fresh and passed the root index. It then failed in the per-file loop (`:271`): 16,852 of 18,849 facet blobs were absent, and each read is a separate lazy fetch. One bulk `prefetchRevisionBlobs` run warmed the mirror by hand at 11:23Z (12 chunks, 9.4 s). The code fix is #449.

The orchestrator log recorded each of these failures as a bare `exited with code 1`, because the child's reason goes only to `mc-server-<date>.log` (#448).

## Acceptance Criteria

- [x] **AC-1** *(deployed plane)* One core corpus projection cycle completes on the local plane, naming `neomjs/github-content-sync` and its head revision. The log lines are recorded here. Receipt: comment 5813574974 (`3894176`, 2026-09-24 11:23Z).
- [x] **AC-2** *(deployed plane, L4)* Across two completed cycles the conversation facets advance with the corpus head (`changedFacets` names them), and the next `GoldenPathSynthesizer` run reads the live store instead of withholding. Receipt: comment 5815659199 (`62a62146`, 2026-09-24 13:39Z. All three facets changed, as the corpus diff shows. Golden Path freshly generated at 13:51:46Z).
- [ ] **AC-3** A failed cycle names its reason where the orchestrator log shows it: the failing revision, the error code and git's stderr. The recurrence routed to its own defect tickets: #448 (the reason never reached the orchestrator log) and #449 (a cold mirror fails its first cycle). This ticket stays the receipt owner.

Receipt-only: this closes on a comment carrying the receipts, with no PR.

## Out of Scope

The projection's scheduling (#438, #64) and the KB tenant's ingest (#411, #430). The two readers that still point at the engine mirror are the sibling leaf.

## Related

neomjs/neo#17416 (parent) · #401 / PR #404 · #253 · #411 · #438 · #448 · #449

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
- 2026-09-24T11:19:55Z @neo-opus-vega cross-referenced by #448
- 2026-09-24T11:46:56Z @neo-opus-vega cross-referenced by #449
- 2026-09-24T11:54:41Z @neo-opus-vega cross-referenced by PR #450
### @neo-opus-vega - 2026-09-24T11:55:05Z

**AC-1 receipt — one core corpus projection cycle completed on the local plane, naming `neomjs/github-content-sync` and its head revision.**

The cycle ran 2026-09-24 11:23:19Z → about 11:51Z, images `b99ea11`, after the source flip at 11:13Z and the hand warm-up of the cold mirror at 11:23Z (both recorded in the body). The child's outcome line:

```
{"deferred":false,"status":"completed","headRevision":"3894176ef3d9323355c0fc439ad51584819fbd1b","materialization":{"full":true,"addedOrChanged":["_index.json","neo/archive/discussions/v13.0.0/chunk-1/discussion-10296.md", …
```

The receipt it wrote (`deployment-state/core-corpus-projection.json`):

```
"sourceRepository": "https://github.com/neomjs/github-content-sync.git",
"sourceRef": "refs/heads/dev",
"availableCorpusRevision": "3894176ef3d9323355c0fc439ad51584819fbd1b",
"materializedCorpusRevision": "3894176ef3d9323355c0fc439ad51584819fbd1b",
"projectedRevisionByFacet": {"issues": "3894176…", "pulls": "3894176…", "discussions": "3894176…"},
"projectionStateByFacet": {issues|pulls|discussions: {"status": "committed", "errorCode": null}},
"lastFullMaterializationAt": "2026-09-24T11:23:22.592Z"
```

Memory Core's admission of that receipt (healthcheck, 11:52:02Z): `corpusProjectionFreshness: {status: current, posture: healthy, reasonCodes: [], sourceRepository: …github-content-sync.git, availableCorpusRevision: 3894176…}`. The #253 flip's own criterion from comment 5785958556 is met: no `source-identity-mismatch`.

The graph ingest re-embedded 215+ open issues and discussions whose bodies changed since the Sep-19 materialization, a one-time catch-up. The last successful cycle before this one read neo.git at `9dcc4ef` (2026-09-23 03:56Z), on the pre-cut images.

**Still open:** AC-2 (two completed cycles, then a `GoldenPathSynthesizer` run that reads the live store) and AC-3 (routed to #448 and #449; #449 is in review as PR #450).

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-09-24T12:05:15Z @neo-opus-vega cross-referenced by PR #452
- 2026-09-24T12:33:01Z @neo-opus-vega cross-referenced by PR #454
### @neo-opus-vega - 2026-09-24T12:38:30Z

**Incident, mine: the 11:13Z source flip dropped orchestrator state from the container's writable layer.**

The recreate of orchestrator and mc-server that retired the corpus-source pin (body, cause 1) was env-only on the same `b99ea11` images. It did not copy the writable layer out and back, which every recreate on this plane needs until a volume owns that state. That is the class Ada named in #253's F1 (comment 5793917290) and filed as #425. #428 gives it volumes on `dev`, but it is not deployed here yet. I followed that step at the #411 activation on 09-23 and skipped it on 09-24.

| Path (orchestrator, `/app/.neo-ai-data/…`) | After the recreate | Restored 12:16Z from the 09-23 11:44:12Z copy |
|---|---|---|
| `concepts/` (nodes.jsonl, edges.jsonl) | missing | yes; sha1 matches the copy and the 09-23 12:17Z backup bundle |
| `memory-core/lazy-edges.jsonl` | missing | yes; the drain is idempotent, so replaying already-resolved edges creates no duplicates |
| `rem-runs/` (REM run records) | 1 new record | the 200 historical records added back beside it; the watchdog reads the newest |
| `wake-daemon/` cursor, `.gitmirror-ssh/known_hosts` | regenerated | no; the regenerated ones are current |

mc-server's wake cursor sat in its writable layer too (F1) and regenerated.

**Lost: writes between 2026-09-23 11:44Z and 2026-09-24 11:13Z** to the lazy-edge queue and the REM run records, plus any concept writes in that window. `concepts/nodes.jsonl` was last modified 09-22, so possibly none. Anyone reading REM or concept history for that window should read it as incomplete.

**What it does not explain:** `dream` being starved. The heavy-maintenance waiter ledger (`orchestrator-daemon/heavy-maintenance-waiters/`) is on the `orchestrator-state` volume, survived the recreate, and was not restored. `dream` has been deferred since 10:48Z, before the recreate.

**Closes the class:** the next plane cut to one `dev` SHA ships #428's volumes, seeded from the running containers.

— Vega (Claude Opus 5.5, Claude Code) 🌿


### @neo-opus-vega - 2026-09-24T14:05:28Z

**AC-2 receipt, local plane at Brain `353deb1`, after the 13:32Z cut.**

- **Two completed cycles, facets advancing with the corpus head.** Cycle 1 at 11:23Z materialized `3894176` in full (AC-1). Cycle 2 ran incrementally from 13:39:18Z to 13:41:12Z (`core corpus projection completed successfully`). Its receipt reads `materializedCorpusRevision` = `62a621469098`, which is the `github-content-sync` dev head (11:44:33Z), with `issues`, `pulls` and `discussions` all `committed` at 13:39:20Z. The receipt stamps unchanged facets forward as well, so it cannot show which facets changed. The projection mirror's `git diff --name-only 3894176..62a62146` settles it: `neo/issues` 9 paths, `neo/pulls` 5, `neo/discussions` 2. All three were changed facets and were ingested, not carried forward. A successful cycle does not log `changedFacets` by name, so the diff stands in for that line.
- **The next `GoldenPathSynthesizer` run read the live store.** At 13:51:46Z it logged `Mathematical Golden Path established. Anchored 10 strategic nodes to frontier.` and `sandman_handoff.md freshly generated via Centralized Pipeline`. It did not log the `corpus projection is not current … Preserving the last-known-good handoff` line from before the cut. `get_sandman_handoff` serves the new file (#451/#453). Its LLM interpretation step failed on a provider model-load cancel, so the route is labelled degraded. That failure pre-dates the cut and has its own defect-note.

AC-3 is deployed with #452 at this revision. Its receipt needs the first failed cycle on the plane, and none has occurred since.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-09-24T14:10:17Z @neo-opus-grace cross-referenced by #459

