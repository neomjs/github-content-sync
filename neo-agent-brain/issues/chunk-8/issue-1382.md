---
id: 1382
title: Tenant sync re-materializes 56k files every 2 min while its embed step fails unclassified
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T21:15:37Z'
updatedAt: '2026-10-31T21:09:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1382'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-10-31T21:09:51Z'
---
# Tenant sync re-materializes 56k files every 2 min while its embed step fails unclassified

## Context

Read for #64 AC-17 on the local plane at Brain `4ca0f506` (cut 17:50Z, 2026-10-30), all six containers healthy, recorded on #64 (comment 6533592892). Instead of two idle sweeps at an unchanged head, `neo-shared/github-content-sync` shows the same four lines every ~2.3 minutes, 62 times between 18:52:36Z and 21:13Z:

```
[INFO] [TenantRepoSync] Refreshing neo-shared/github-content-sync (embedding recovery generation).
[INFO] [TenantRepoSync] neo-shared/github-content-sync materialized: envelopeFiles=56524 envelopeDeleted=0 ingested=56586 deleted=0 embeddings=0 errors=1
[WARN] [TenantRepoSync] neo-shared/github-content-sync deferred: embedding incomplete, checkpoint held at 67be3a09 codes=KB_VECTOR_EMBED_FAILED ingested=56586 embeddings=0 (streak held at 1)
[INFO] [TenantRepoSync] neo-shared/github-content-sync suppressed by backoff (next ~+1 h, consecutiveFailures=1, backoffX=2, lastErrorCode=KB_TENANT_REPO_SYNC_SYNC_FAILED).
```

The first `KB_VECTOR_EMBED_FAILED` is at 18:51:17Z, the first sweep after the cut's backoff; no run since has `embeddings > 0`. Neither the orchestrator nor the kb-server log carries an `[ERROR]` line or exception text for the `errors=1`. The host's LM Studio answers `/v1/models` with `google/gemma-4-26b-a4b` and `text-embedding-qwen3-embedding-8b` throughout, so the canary that opens a recovery generation sees a healthy provider.

## The Problem

Three defects compound into a plane that burns a full 56,524-file materialization every two minutes and never advances its checkpoint:

1. **The recovery bypass re-opens on every sweep.** The design (TenantRepoSyncService.mjs, the `embeddingRecovery` episode around :3005) requires "a newly healthy canary generation before another bypass, so a still-starved provider cannot buy one retry per sweep". The canary is healthy every time; the real embed batch fails every time; the gate opens 62 times. The backoff's "next ~+1 h" is printed and not honoured.
2. **A recovery generation redoes the whole materialization.** The checkpoint is held at `67be3a09` and the envelope is unchanged (`envelopeFiles=56524 envelopeDeleted=0`), yet each generation re-ingests 56,586 records before reaching the one step that failed.
3. **The failure is unclassified and unlogged.** `KB_VECTOR_EMBED_FAILED` is `KB_VECTOR_EMBED_UNCLASSIFIED` (`embedFailureClassification.mjs:67`): the provider's code matched neither vocabulary, and the counter `errors=1` is the only trace. Nobody can tell from the plane what the provider said.

## The Architectural Reality

- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs`: due/backoff decision and the `suppressed by backoff` line (:2560–2590), the refresh with `recoveryGrant` consuming `embeddingRecovery.bypassConsumedAt` (:2722–2734), the deferred branch building the recovery episode from `deferredCauseCode` (:3005–3030); `TenantRepoSyncErrors.mjs` beside it; `ai/daemons/orchestrator/scheduling/tenantRepoSync.mjs` schedules it (structure map: `ai/daemons/orchestrator/services`, `ai/daemons/orchestrator/scheduling`).
- `ai/services/knowledge-base/helpers/embedFailureClassification.mjs`: the bounded `KB_*` translation at the boundary; an unrecognised provider code becomes `KB_VECTOR_EMBED_UNCLASSIFIED` and defers by design (:400–420). The design is right about deferring; it says nothing about how often a deferral may re-open the gate, and the unclassified case leaves no evidence behind.
- `ai/services/knowledge-base/IngestionService.mjs:663` is where the `errors` counter is produced.
- Decision Record impact: aligned-with the self-healing direction (ADR 0025 diagnose, 0026 actuate): the plane keeps retrying; the defect is the retry's cost and its silence.
- Design authority: neomjs/neo#16692, the recovery contract this gate implements. Its AC-1 promises that a recovered dependency re-arms a suppressed repo "without waiting out the full `effectiveCadenceMs`"; its AC-4 that "a recovery signal that is itself unreliable must not defeat the protection". This ticket narrows AC-1 under AC-4, deliberately. A recovery generation that repeats its episode's outcome proves the canary unreliable for that episode, so the episode's one bypass is spent and recovery waits for the ordinary cadence (about 2 h on the plane). The spent bypass survives an embedding-class cause-code change, because `buildEmbeddingRecoveryEpisode` keeps `episodeId` and the consumed history whenever a prior episode exists. Complete success drops the episode; a non-embedding failure clears it only once work has started, and a non-embedding deferral keeps it. **Accepted cost:** after a real provider fix, a repo whose episode already spent its bypass waits for its ordinary retry instead of re-arming at once.

## The Fix

1. The recovery gate opens once per healthy canary **and** a failed recovery generation closes it for the normal backoff interval; a canary that passes while the batch fails is a signal to shrink the canary/batch gap, not to retry at once.
2. ~~A recovery generation at a held checkpoint re-runs the embed step for the held generation only; it does not re-materialize an unchanged envelope.~~ **Deferred (2026-10-31):** once fix 1 lands, a recovery generation runs at most once per backoff interval (about 2 h on the plane) instead of about every 2 min. The per-attempt cost is re-measured on the plane after that, and gets its own ticket only if it still matters.
3. **Moved to #1403 (2026-10-31).** ~~The unclassified path logs the provider's stage and a bounded summary of the failing request (count of failing documents, the longest document's token count under the 32k embedding window) at `WARN`, and the bounded code names the stage (`KB_VECTOR_EMBED_UNCLASSIFIED_<stage>`) so the next reader does not need this ticket.~~

## Acceptance Criteria

- [ ] A unit arm reproduces the loop: healthy canary + failing batch yields at most one recovery generation per backoff interval; red on `dev` first.
- [ ] ~~A unit arm: a recovery generation at a held checkpoint with an unchanged envelope performs zero materialization work and one embed attempt.~~ Deferred with fix 2.
- [ ] ~~An unclassified embed failure writes one `WARN` with the stage and the bounded summary; the deployment-state snapshot carries the stage-qualified code.~~ Moved to #1403.
- [ ] **Post-merge (L3), deferred to #64:** installed receipt on the plane after the cut: while the embed step still fails, the orchestrator log shows at most one `github-content-sync` attempt per backoff interval. The checkpoint advancing waits on the cause, which #1405 found and fixes.

## Out of Scope

The embed failure's cause, which is #1405: a corpus record holding U+2028 split in the KB embed reader, so the batch never reached the provider. #64's own scheduling-starvation ACs and #65's blobless-mirror cost.

## Related

Parent: #64. Split on 2026-10-31: #1403 carries the trace (fix 3); fix 2 is deferred above. #65 (first-ingestion round trips). #1362 / #1352 (the cut's intended fixes, unconfirmable while this loop runs). A2A defect-note fingerprint `1fcc82b9d92eb05f`.

Live latest-open sweep: latest 20 open Brain issues read at 2026-10-30T21:14:07Z; no equivalent (#64, #65, #54 are the neighbours, none this loop). A2A sweep: last 30 `lane-claim` messages, none on TenantRepoSync. MC sweep: "github-content-sync embedding incomplete KB_VECTOR_EMBED_FAILED recovery generation re-materialize every two minutes backoff bypassed", 6 results, no prior decision (2026-09-28 Emmy's stale-Golden-Path exploration is adjacent, not this). Own-assignment sweep: 10 open Brain tickets, #64 is the parent, #23 (embedding lane consolidation) is adjacent by substrate and does not cover the loop. Structure map: `ai/daemons/orchestrator/services` owns the sync, `ai/services/knowledge-base/helpers` the classification.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "tenant sync embedding recovery generation every two minutes 56524 files KB_VECTOR_EMBED_FAILED unclassified"



## Timeline

- 2026-10-30T21:15:37Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T21:15:39Z @neo-opus-vega added the `bug` label
- 2026-10-30T21:15:39Z @neo-opus-vega added the `ai` label
- 2026-10-30T21:15:39Z @neo-opus-vega added the `performance` label
- 2026-10-30T21:15:40Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T21:15:47Z @neo-opus-vega added parent issue #64
- 2026-10-30T21:16:12Z @neo-opus-vega cross-referenced by #64
- 2026-10-31T20:14:28Z @neo-opus-vega cross-referenced by #1403
- 2026-10-31T20:15:42Z @neo-opus-vega cross-referenced by PR #1404
- 2026-10-31T20:33:56Z @neo-opus-vega cross-referenced by #1405
- 2026-10-31T20:45:31Z @neo-opus-vega referenced in commit `2d27d3e` - "docs(orchestrator): the checkpoint normalizer no longer promises a canary re-arm after a spent bypass (#1382)

Canonicalizing a consumed generation folds it into lastConsumedGenerationId, and since
the awaiting-recovery cohort refuses that history, a restart no longer re-arms the canary
or rotates its key: only the ordinary backoff retries the repo.

The ticket archaeology guard reads a touched file whole, so the file's two remaining
comment references move here: the terminal-stop allowlist entry was measured in #238
round 2, and the clean-partial resume marker came from #430."
- 2026-10-31T20:49:56Z @neo-gemini-julia cross-referenced by PR #1406
### @neo-opus-vega - 2026-10-31T20:56:48Z

**Handover (Vega, session sunset 2026-10-31 ~21:00Z): PR #1404 is APPROVED in round 2 and at the operator's merge gate.** Sophie approved at exact `2d27d3e8` after the single RA was addressed: design authority against neomjs/neo#16692, the L3 deferral to #64, and the stale normalizer comment. CI is 23/23 and the PR is CLEAN.

Pickup after the merge: the installed AC (deferred to #64) needs a still-failing embed. #1406 (#1405) removes the current failure, so if both land in one cut the loop condition disappears. The receipt then either waits for the next persistent embed failure, or cites the unit arm as the evidence and says so on #64.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-10-31T21:09:51Z @tobiu referenced in commit `c3f9b51` - "fix(orchestrator): a spent embedding-recovery bypass leaves retries to the ordinary backoff (#1382) (#1404)

* fix(orchestrator): a spent embedding-recovery bypass leaves retries to the ordinary backoff (#1382)

A recovery generation that deferred again folded back into its episode
under a new consumption key, so the next sweep ran a fresh canary, got a
healthy answer and minted another generation. On the canonical plane that
was a full 56k-file re-materialization every ~2 min with embeddings=0.
A canary that passes while the batch fails is no evidence, so an episode
now gets one bypass and the ordinary backoff retries after it.

The #16692 spec that pinned the immediate re-arm now pins the spent bypass
and the ordinary retry.

* docs(orchestrator): the checkpoint normalizer no longer promises a canary re-arm after a spent bypass (#1382)

Canonicalizing a consumed generation folds it into lastConsumedGenerationId, and since
the awaiting-recovery cohort refuses that history, a restart no longer re-arms the canary
or rotates its key: only the ordinary backoff retries the repo.

The ticket archaeology guard reads a touched file whole, so the file's two remaining
comment references move here: the terminal-stop allowlist entry was measured in #238
round 2, and the clean-partial resume marker came from #430."
- 2026-10-31T21:09:52Z @tobiu closed this issue
- 2026-10-31T21:27:23Z @neo-opus-vega cross-referenced by #1411

