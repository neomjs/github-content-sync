---
id: 411
title: Activate the github-content-sync KB tenant on the deployed plane
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-22T22:25:56Z'
updatedAt: '2026-09-23T12:37:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/411'
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
blockedBy:
  - '[x] 237 A ref-not-found is retried as a transient, 36 times and counting'
blocking: []
closedAt: '2026-09-23T11:32:35Z'
---
# Activate the github-content-sync KB tenant on the deployed plane

## Context

#402 delivers the code half of the operator's top item (2026-09-21: `ask_knowledge_base` and `query_documents` unusable, stale for a month): the `ConversationCorpusSource` extractor and the tenant declaration for `neomjs/github-content-sync` in `deploy/cloud/kb-config.yaml`. That declaration ships `disabled: true`. This ticket is the activation half — the plane-side acceptance criteria no PR can satisfy from inside a sandbox — split out of #402 so the code leaf can close on its code.

> **Status 2026-09-23T11:2xZ:** the #253 cut is done — the plane runs `b99ea11` (11:03Z), lossless. Post-cut snapshot (11:04Z): `tenantRepoSync.enabled: false` (both sync controls off, as #253 designed), 5 repos with the corpus entry present as `disabled: true`, #237's specimen unchanged at `backoff-suppressed` / 250 because the lane has not evaluated it. #282 merged (PR #423, 10:26Z). The flip is PR #424 (Round 1 by @neo-gpt: two RAs on stale post-cut facts and the residual owner — addressed). Per the Evidence Ladder, AC-1's observation and AC-3/AC-4 are `[L4-deferred — operator handoff needed]` residuals owned by #64 AC-6 and recorded here after the activation transaction.
>
> **Status 2026-09-23T12:1xZ:** PR #424 merged 11:32Z as `75a50fc`; the activation transaction ran 11:44–11:45Z (@neo-opus-ada). First enabled sweep read 11:54Z: AC-1's specimen recovered instead of stopping (receipt comment below; #237 closed on it, the stop arm re-homed to #64 AC-7); AC-3 in progress — two slices so far (11:48–11:54Z, 12:20–12:25Z): 47,182 chunks ingested each time, 120 then 140 embedded, 46,922 outstanding at 12:25Z, checkpoint held at `null`; at ~140 embeddings per 30-minute cycle the checkpoint, and AC-4 behind it, are ~5–7 days out at the shipped knobs (defect-note 12:27Z; the scheduling leaf is #430 under #64).

## The Problem

The tenant poller (`tenant-repo-sync`) is not healthy. #237's specimen `453ffb0965b3` has retried a ref-not-found as a transient 215+ times (2026-09-19 read) with `accessReadiness: degraded`, and at 2026-09-22T21:59Z the lane holds the lease and defers `core-corpus-projection`, `dream` and `graphlog-compaction` (@neo-gpt-emmy's read-only healthcheck). A tenant enabled into that state inherits it, and its failure would read as the extractor's. The deployed plane also still runs Engine revision `467fd122` (2026-08-25, pre-split), so nothing merged into the Brain reaches it until #253's cut.

## The Architectural Reality

- `deploy/cloud/kb-config.yaml` — tier 2 of tenant-config resolution, mounted read-only into the orchestrator and kb-server (`deploy/cloud/docker-compose.local-agent-os.yml:48`, `:89`; documented for the cloud compose at `docker-compose.yml:529-536`). The `github-content-sync` entry carries the extraction profile and `disabled: true`.
- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs:312` — `repo.disabled === true` skips the entry. Removing the flag is the whole activation *on the config side*; the lane's master toggle is a separate producer.
- `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED` — the lane's master toggle: cloud profile default-on when `tenantRepos[]` is configured, local profile default-off (`learn/agentos/DeploymentCookbook.md`). It read `true` on the pre-cut plane (four tenant repos syncing) and the #253 cut left it `false`; switching it on is an operator step of the activation transaction, not a compose change.
- Freshness surfaces: `IngestionService.getTenantManifest({tenantId, repoSlug})` (the corpus-owned manifest), the extraction receipt bound to a `github-content-sync` revision, and `ask_knowledge_base` answers that cite conversations with their origin.

## The Fix

1. Disposition #237's specimen — the owner's waiver on the deployed fix (comment 5793702894): the retry loop no longer exists in `b99ea11`'s code, so the new entry inherits nothing; the observed stop is the first enabled sweep's receipt and closes #237 then.
2. One-line PR (#424): remove `disabled: true` from the `github-content-sync` entry.
3. The activation transaction, one recreate: runtime root at the merged head (the mounted `kb-config.yaml` carries the flip), `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED=true`, recreate kb-server + orchestrator, wait one sweep, record the receipts here and on #64 AC-6 in order AC-1 → AC-3 → AC-4.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `deploy/cloud/kb-config.yaml`, `github-content-sync` entry | tenant-repo access contract (`normalizeTenantRepoEntry` spreads the entry; `disabled` read at `TenantRepoSyncService.mjs:312`) | flag removed → the entry is eligible; the poller mirrors and ingests the corpus on its first sweep after the lane's master toggle is on | flag present → no ingestion; never a fallback to the Engine tree | `learn/agentos/cloud-deployment/TenantIngestionModel.md` | AC-3 manifest + receipt |

## Decision Record impact

`aligned-with` D#17846 §8.6 / §8.6a / §8.7 and ADR 0019; nothing amended. Depends on #237 (native `blocked_by`).

## Acceptance Criteria

- [x] **AC-1** — #237's `453ffb0965b3` specimen is dispositioned before the flip: the ticket is closed, or its owner records a waiver here naming what the new tenant inherits. *(Waiver recorded 2026-09-23 11:05Z, comment 5793702894: it inherits nothing — the deployed code cannot retry an unresolvable ref. The observed `stopped-unresolvable-ref` is `[L4-deferred — operator handoff needed]` to the first enabled sweep and closes #237; residual owner #64 AC-6.)* **Observed 11:54Z, not as planned:** the specimen (`neo-shared/devindex`) recovered instead of stopping — `completed: head=6e7fcc72 ingested=154 deleted=0`, `consecutiveFailures 250 → 0` — because the #253 cut gave the lane its first config carrying `65b0a21`'s `branchRef main → dev` (#267; the pre-split plane received no Brain merges, and the mirror's fetches had succeeded throughout). The tenant inherited nothing, as the waiver said; #237 is closed on the recovered specimen, and the stop arm's live receipt is re-homed to #64 AC-7 (receipt comment below).
- [x] **AC-2** — `disabled: true` is removed from the `github-content-sync` entry (PR #424), and the local Agent OS profile has the tenant-sync toggle on or a documented operator step. *(Documented: the toggle read `true` pre-cut and `false` post-cut; `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED=true` is step two of the activation transaction on this plane and on any fresh local install.)* The PR `Resolves` this ticket; AC-1's observation and AC-3/AC-4 are its declared residuals. **Done:** PR #424 merged 11:32Z as `75a50fc`; the toggle flipped 11:44Z (@neo-opus-ada); snapshot 11:54Z `tenantRepoSync.enabled: true`, 5 repos, `disabledCount 0`.
- [ ] **AC-3** — *(deployed plane)* `[L4-deferred — operator handoff needed]` a corpus-owned manifest (`tenantId: 'neo-shared'`, `repoSlug: 'github-content-sync'`) exists and the extraction receipt is bound to a `github-content-sync` revision — recorded here and on #64 AC-6 before any freshness claim.
- [ ] **AC-4** — *(deployed plane, after AC-3)* `[L4-deferred — operator handoff needed]` `ask_knowledge_base` cites a `neomjs/neo` conversation created the same day and a `neo-agent-brain` conversation, each showing its origin.
- [ ] **AC-5 — ordering and the old-row boundary** (added 2026-09-22 from @neo-gpt's #412 review preflight; re-folded the same night on his #282 read). The three retired Sources leave the legacy `kbSync` path emitting no conversation chunks, and `VectorService.embed()`'s default stale strategy (`delete-upfront`, scoped to `neo-shared/neo`) retires whatever an earlier sync stamped there — **including source-code rows if the legacy `ApiSource` scan runs against the post-cut hierarchy**. So: **the legacy `kbSync` stays OFF through this ticket's activation** — #253 leaves both sync controls off, and this ticket flips only the tenant sync. #282 (merged 2026-09-23) re-enables the core scan additively (`deleteStale: false`, explicit strategy refused); the retirement of the frozen 2026-08-26 `neo`-owned conversation rows is owned by **#417** — a scoped delete by `{repoSlug: 'neo', type ∈ conversation types}` **with a source-code-row preservation control** — not by this ticket and never by a legacy sync. AC-3 and AC-4 are therefore verified while the stale `neo`-owned conversation rows coexist with the fresh corpus-owned ones — a named coverage boundary: `ask` may still surface a frozen row beside a fresh one until #417 lands, and that is recorded here rather than hidden.

## Out of Scope

- The extractor, the declaration and the retirement of the three Engine-tree Sources — #402.
- The poller's starvation and the blobless-mirror cost — #64, #65.
- The image cut — #253.

## Avoided Traps

- ⛔ Do not enable the tenant "to see whether it works" while #237's lane is in its failure state: the new entry inherits the back-off and the lease contention.
- ⛔ Do not claim freshness from a green sweep — or from the flag removal alone: the flag and the lane's master toggle are separate producers, and post-cut the toggle is off. The manifest and the revision-bound receipt are the evidence; `ask` citing today's conversation is the outcome.
- ⛔ Do not keep the PR at `Refs` to hold the ticket open: the PR-body guard refuses it, and a residual owner (#64 AC-6) is the ladder's shape for deployed receipts.

## Related

#402 (code leaf) · #237 (readiness gate) · #253 · #64 (residual owner, AC-6) · #65 · #246 / PR #410 (the FM feed over the same corpus) · #282 / PR #423 (additive core profiles) · #417 · #419 · neomjs/neo#17416 (epic) · D#17846 §8.6, §8.7

Live latest-open sweep: latest 20 open Brain issues at 2026-09-22T22:15Z — none equivalent (nearest #237, #64, #253). A2A in-flight sweep (30 most recent, all read-states, 22:26Z): no claim on tenant activation; #402 ownership confirmed by @neo-gpt-emmy (21:57Z) and @neo-fable-clio (22:01Z). Memory Core sweep: no prior decision beyond D#17846 §8.7 step 1. Own-assignment sweep: #402, #362, #237, #23, #64, #65 — none equivalent. Structure map: N/A — no `.mjs` placement; one config line under `deploy/cloud/`.

Origin Session ID: fc04c361-0cae-4a80-9506-fa2ef4785d2b
Retrieval Hint: "github-content-sync tenant activation disabled flag kb-config deployed plane freshness receipt #237 specimen"

> **Update 2026-09-23 ~10:55Z (own body, in place):** AC-2 and AC-3/AC-4 restated to the Evidence Ladder shape (PR `Resolves`, deployed receipts `[L4-deferred]` with #64 as residual owner); #282's merge folded into AC-5.
> **Update 2026-09-23 ~11:2xZ (own body, in place; PR #424 Round-1 RA-1/RA-2):** post-cut facts corrected — the toggle is off after the cut, not on; AC-1 carries the owner's waiver with the observation deferred; the residual owner is #64's new AC-6, which names the manifest, the same-day ask and the forecast explicitly.



## Timeline

- 2026-09-22T22:25:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-22T22:25:58Z @neo-opus-vega added the `enhancement` label
- 2026-09-22T22:25:58Z @neo-opus-vega added the `ai` label
- 2026-09-22T22:25:59Z @neo-opus-vega added the `agent-os` label
- 2026-09-22T22:26:31Z @neo-opus-vega marked this issue as being blocked by #237
- 2026-09-22T22:28:31Z @neo-opus-vega cross-referenced by #402
- 2026-09-22T22:30:07Z @neo-opus-vega cross-referenced by PR #412
- 2026-09-22T22:48:39Z @neo-fable cross-referenced by #19057
- 2026-09-22T22:50:21Z @neo-fable cross-referenced by #19058
- 2026-09-22T23:29:12Z @neo-opus-vega cross-referenced by #415
- 2026-09-22T23:39:58Z @neo-opus-vega cross-referenced by #417
- 2026-09-22T23:45:42Z @neo-gpt cross-referenced by #282
- 2026-09-23T00:19:56Z @neo-opus-ada cross-referenced by #253
- 2026-09-23T00:49:59Z @neo-gpt-emmy cross-referenced by PR #418
- 2026-09-23T01:34:40Z @neo-gpt cross-referenced by #419
- 2026-09-23T01:41:10Z @neo-opus-vega cross-referenced by #420
### @neo-opus-vega - 2026-09-23T02:29:35Z

**Session handover (sunset 2026-09-23 ~02:30Z) — owner @neo-opus-vega; the lane stays mine.**

**State:** the extractor (#402) is merged — PR #412, dev@72fc142 — and the `github-content-sync` entry in `deploy/cloud/kb-config.yaml` sits at `disabled: true`. AC-5 was re-folded after @neo-gpt's #412 review: the legacy `kbSync` stays OFF through activation, and the frozen `neo`-owned conversation rows retire under #417 (after #282), never here. Corpus tip at sunset: github-content-sync dev@df98ae56 (01:31Z publication); the scheduled publisher is green on every run, delivered every 2.7–5.5 h.

**Gate:** the #253 cut (owner @neo-opus-ada).

**Pickup protocol, after the cut:**
1. Read `healthcheck` on the container plane — `tenantRepoSync` must show #237's ref reported as `stopped-unresolvable-ref`, not retried (the fix ships with the cut).
2. Enable: drop `disabled: true` from the entry; locally `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED=true`; recreate kb-server + orchestrator only (chroma and MC untouched).
3. Record the first cycle summary and the tenant manifest revision here as AC-3 / AC-4 receipts.
4. AC-4's coexistence boundary (fresh corpus rows beside frozen legacy rows) stays open until #417 runs.

Nothing is in flight on a branch; no local state to recover.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-09-23T02:45:21Z @neo-gpt-emmy cross-referenced by PR #19068
- 2026-09-23T08:59:30Z @neo-opus-vega added parent issue #17416
- 2026-09-23T10:20:02Z @neo-opus-vega cross-referenced by PR #423
- 2026-09-23T10:55:20Z @neo-opus-vega cross-referenced by PR #424
- 2026-09-23T11:02:37Z @neo-opus-vega cross-referenced by #105
### @neo-opus-vega - 2026-09-23T11:05:35Z

## AC-1 disposition — the fix is deployed; the stop becomes observable when the lane is switched on

Read from `get_deployment_state_snapshot` at 11:04Z, 13 s old, on the post-cut plane (`deployedRevision b99ea11c21`, `healthcheck` 11:03Z, uptime 4 s):

| field | pre-cut `467fd122` (10:47Z) | post-cut `b99ea11` (11:04Z) |
|---|---|---|
| `tenantRepoSync.enabled` | `true` | **`false`** — #253 left both sync controls off, as designed |
| repos | 4 | 5 — the `github-content-sync` entry is present as `status: disabled`, `disabled: true`, exactly what rehearsal 4 wanted the cut to carry |
| specimen `453ffb0965b3` | `backoff-suppressed`, failures 250, `due: true` | unchanged — the lane has not evaluated it, because the lane is off |

So AC-1's observation (`stopped-unresolvable-ref`) cannot precede the activation: it needs one lane evaluation, and the lane is disabled until the activation transaction switches it on. **Owner's disposition, in AC-1's own terms:** the new tenant inherits nothing from #237's specimen, because the retry loop no longer exists in the deployed code — `stopped-unresolvable-ref` is on the plane (`TenantRepoSyncService.mjs:2343` / `:3180`, commits `eebd6e3` / `c11ba00` / `cb037fd`, all in `b99ea11`). The first evaluation after `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED=true` stops the specimen instead of retrying it; #237 closes on that read, recorded there and here.

**Activation transaction (one recreate, not two):** merge PR #424 (drops `disabled: true`), advance the plane's runtime root to the merge commit so the mounted `kb-config.yaml` carries it, set the tenant-sync toggle on, recreate kb-server + orchestrator, wait one sweep. Receipts then land here in this order: #237's stop (AC-1), the tenant manifest + revision-bound extraction receipt (AC-3), `ask_knowledge_base` citing same-day conversations (AC-4). The host steps are @neo-opus-ada's plane mechanics — asked on A2A once the write freeze ends.

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-09-23T11:23:35Z @neo-opus-vega referenced in commit `71e6323` - "docs(kb): the corpus tenant comment names the activation transaction (#411)"
- 2026-09-23T11:24:04Z @neo-opus-vega cross-referenced by #64
- 2026-09-23T11:29:47Z @neo-opus-ada cross-referenced by #425
- 2026-09-23T11:32:35Z @tobiu referenced in commit `75a50fc` - "Merge pull request #424 from neomjs/vega/411-activate-corpus-tenant

feat(kb): activate the github-content-sync tenant (#411)"
- 2026-09-23T11:32:35Z @tobiu closed this issue
- 2026-09-23T11:38:53Z @neo-gpt-emmy cross-referenced by #426
### @neo-opus-vega - 2026-09-23T12:07:42Z

## Activation receipt 1 of 3 — AC-1, as observed (not as planned)

**The transaction** (executed by @neo-opus-ada 11:44–11:45Z): runtime root `75a50fc`, images `b99ea11`, `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED=true` the only flip (`KB_SYNC` and `PRIMARY_DEV_SYNC` false), kb-server + orchestrator recreated under the F1 precondition (writable-layer state intact, 136/182 concepts). Snapshot 11:54:01Z: `tenantRepoSync.enabled: true`, `status: running`, 5 repos, `disabledCount: 0`; first sweep `5 repos, 2 completed, 0 failed, 1 partial-progress, 2 revalidation-deferred`.

**AC-1 — #237's specimen `453ffb0965b3` (`neo-shared/devindex`).** Planned receipt: `stopped-unresolvable-ref` on the first enabled evaluation. Observed: the lane attempted it at 11:44:23Z and it **completed** — orchestrator log `neo-shared/devindex completed: head=6e7fcc72 ingested=154 deleted=0 (262081ms)`; snapshot `status: active`, `lastIngestedRev 6e7fcc72ddc9`, `consecutiveFailures 250 → 0`, `stopReasonCode null`, `recoveryState null`. The ref that never resolved was `main`: the mirror inside the container has no `refs/heads/main`, its fetches succeeded through the whole window (PR-ref files dated 2026-08-26 … 2026-09-23 03:36Z), and kb-config `65b0a21` (#267, 2026-08-31) had moved devindex to `branchRef: dev` — but the pre-cut plane ran the pre-split Engine image (`467fd122`), so no Brain merge, that fix included, reached the lane until the #253 cut; this sweep was its first evaluation under the fixed config. The input changed in the same transaction that deployed the stop, so the resume-on-input-change arm ran live and the stop arm had no specimen.

Consequences: the waiver's claim holds — the new tenant inherited nothing (the sweep that first evaluated it had a clean cohort). #237 is closed on the recovered specimen; the stop arm's live receipt is re-homed to #64 AC-7 (first natural specimen, no synthetic entry — a `branchRef` mutation on the shared plane is operator-owned).

**AC-3 — in progress, not a receipt.** The corpus entry's first slice, 11:48:45–11:54:01Z (after kb-server's 11:45:39Z re-bind): `materialized: envelopeFiles=47140 envelopeDeleted=0 ingested=47182 deleted=0 embeddings=120 errors=0` → `partial-progress: slice budget reached, checkpoint held at none`. Snapshot: `lastIngestedRev null`, `corpusOutstanding.remaining 47062`, `checkpointStatus: failed` (the derived label for "attempted at contract v2, no rev yet" in `classifyTenantRepoCheckpoint`, not a write failure). No manifest and no revision-bound receipt exist yet; the entry is due again 12:19Z. AC-4 follows AC-3.

Provenance for neomjs/neo-agent-brain#426: activation 11:44–11:45Z · runtime root `75a50fc` · images `b99ea11` · corpus `dev` head at first ingest `267fedfd` (2026-09-23T07:46:03Z) · devindex head `6e7fcc72`.

— Vega (Fable 5.1, Claude Code) 🌿


- 2026-09-23T12:07:47Z @neo-opus-vega cross-referenced by #237
- 2026-09-23T12:32:50Z @neo-opus-vega cross-referenced by #429
- 2026-09-23T12:33:27Z @neo-opus-vega cross-referenced by #430
- 2026-09-23T12:52:42Z @neo-opus-vega cross-referenced by #432
- 2026-09-23T13:18:51Z @neo-gpt-emmy cross-referenced by PR #107
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434

