---
id: 430
title: The corpus tenant's first ingest lands one slice of embeddings per 30-minute cadence and rebuilds its 47k-file envelope every time
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T12:33:26Z'
updatedAt: '2026-09-23T14:59:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/430'
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
# The corpus tenant's first ingest lands one slice of embeddings per 30-minute cadence and rebuilds its 47k-file envelope every time

## Context

Leaf of #64 (its scheduling-fairness half). Found while reading the activation receipts of neomjs/neo-agent-brain#411 (PR #424, activated 2026-09-23 11:44Z): the `github-content-sync` tenant is the first corpus on the local plane whose first ingest does not fit one slice, and the lane's shape for that case was designed for fairness among small repos, not for a 47k-chunk first ingest.

## The Problem

Two consecutive slices of the corpus tenant on `neo-local-canonical` (`b99ea11`), from the orchestrator log and `get_deployment_state_snapshot`:

| slice | window | materialized | ingested | embeddings | outcome |
|---|---|---|---|---|---|
| 1 | 11:48:45–11:54:01Z | `envelopeFiles=47140` | 47,182 | 120 | `partial-progress`, checkpoint held at `null` |
| 2 | 12:20:01–12:25:10Z | `envelopeFiles=47140` | 47,182 | 140 | `partial-progress`, checkpoint held at `null` |

`corpusOutstanding.settled 120 → 260`, `remaining 46,922`, `nextDueAt 12:50:35Z`. Three facts compound:

1. **Every slice rebuilds the whole envelope.** `lastIngestedRev` stays `null` until the corpus is exhausted, so `buildIngestEnvelope` takes the full path each cycle: 47,140 files materialized and 47,182 chunk rows upserted before the first embedding batch runs. The work that does not need repeating is repeated on every cycle.
2. **A clean partial slice waits a full cadence.** The `partial-progress` branch stamps `lastRunAttemptAt: startedMs` and nothing else (`TenantRepoSyncService.mjs:2902–2915`), so `isRepoDue` re-admits the repo after `effectiveCadenceMs` (`intervals.tenantRepoSyncMs` 30 min + jitter), not at the next 60-second sweep. The log's "repo due next cycle" is the cadence cycle.
3. **The plane's embedding throughput is the ceiling inside the slice.** devindex landed 154 chunks in 262 s including clone and materialization; the corpus slices land 120–140 in the remainder of a 5-minute budget after the rebuild.

Together: ~140 embeddings per 30 minutes, ≈5–7 days to the first checkpoint at the shipped knobs (`sliceBudgetMs` 5 min, `sweepCadenceMs` 60 s, `intervals.tenantRepoSyncMs` 30 min), and the heavy-maintenance lease held 5 of every 30 minutes for a lane that is mostly redoing materialization. Until the checkpoint lands, `ask_knowledge_base` cannot cite a corpus row, which is the outcome neomjs/neo#17416 cornerstone 1 (KB currency) exists for. The KB collection count is flat between slices by construction, so it is not the instrument; `corpusOutstanding` is.

## The Architectural Reality

- `sliceBudgetMs` is a **safe-point** budget checked between embedding batches (`configBase.mjs:2313–2340`), introduced so a concurrency slot is released on progress rather than on corpus completion — measured against sibling repos starved 4+ hours behind one backlog. It does what it was built for; a first ingest larger than one slice is the case it was never sized against.
- `partial-progress` is by construction a run in which nothing failed (`TenantRepoSyncService.mjs:2865–2975`): streak cleared, no recovery episode, checkpoint held. The comment block already says the repo needs "another turn, not a diagnosis"; the scheduler gives it that turn only after the global cadence.
- The ingestion path is idempotent on content-hashed chunk ids (already-embedded chunks are skipped, so each slice resumes where the last stopped) — resumption is correct; only the cost per resumption is wrong.
- The lease is released between slices (`heavyMaintenanceStarvation.leaseHolder: null` at 12:16Z), so waiters do run between cycles; a shorter re-admission for a clean partial repo must keep that property, which is what the yield primitive already guarantees at each safe point.
- **Fairness is enforced at lease acquisition, not by the tenant cadence, and only within a rank.** `MaintenanceBackpressureService` asks `findWaiterToYieldTo` before every acquisition (`heavyMaintenanceWaiterLedger.mjs`). A higher-rank waiter wins immediately, a same-rank waiter wins once it has starved past `fairnessYieldAfterMs` (30 min, `NEO_HEAVY_MAINTENANCE_LEASE_FAIRNESS_YIELD_MS`) and waited longer than the acquirer, and a lower-rank waiter never wins. A first ingest keeps a null `lastIngestedRev` until the whole corpus has landed, so `isBootstrapCriticalTask` ranks it above ordinary maintenance, and re-admitted every sweep it would hold the lease for the whole first ingest. Lever 1 therefore needs the bootstrap class to cover only the first slice. Once a clean slice has landed, the catch-up ranks ordinary and same-rank fairness hands `dream`, `core-corpus-projection` and `graphlog-compaction` their turns on the ledger's clock.
- `isRepoDue` is a pure function (`scheduling/tenantRepoSync.mjs`): `due = recoveryBypass || (now − lastRunAttemptAt) ≥ effectiveCadence`. A `partial-resume` reason beside `embedding-recovery` is the smallest seam for lever 1; the marker rides the persisted repo state, so the checkpoint allowlist has to admit it or a reload drops it silently.

## The Fix

Two levers, both inside the lane; implementer's choice of order, ACs cover the outcome:

1. **Re-admit a clean partial repo at the next sweep, not the next cadence.** A `partial-progress` outcome with `errors=0` records a resume marker (e.g. `partialProgressAt` beside `lastRunAttemptAt`) and `isRepoDue` treats a repo carrying it as due once the sweep cadence has elapsed. Failures keep the existing backoff; `complete` clears the marker. The heavy-maintenance waiters keep their turn through the existing between-slice lease release.
2. **Do not rebuild an envelope whose head has not moved.** Persist the materialized head (`partialHead`) in the checkpoint state (`normalizeTenantRepoCheckpointState` allowlist, validated whole or dropped) and, when the mirror's head equals it, skip materialization and the row upsert and go straight to the outstanding chunks. A moved head invalidates the marker and rebuilds as today.

Operator knob for the meantime, documented not defaulted: `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_SLICE_BUDGET_MS` raised for a first ingest, accepting the longer lease hold while it runs.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `isRepoDue` for a repo in `partial-progress` | `TenantRepoSyncService` scheduler | due after `sweepCadenceMs` when the last slice was clean | absent marker → today's `effectiveCadenceMs` | unit arm: clean partial → due next sweep; failed slice → unchanged backoff |
| persisted `partialProgressAt` / `partialHead` | `normalizeTenantRepoCheckpointState` allowlist | survive reload; validated whole or dropped; cleared on `complete` | `null` — a half-record cannot shorten a cadence or skip a rebuild | mutation: removing either from the allowlist reddens its witness |
| envelope build on an unchanged head | `tenantRepoIngestEnvelopeBuilder` / `TenantRepoSyncService` | reuse the materialization, embed outstanding chunks only | head moved or marker absent → full rebuild as today | log receipt names the reuse; `envelopeFiles` not re-materialized |
| operator runbook | `learn/agentos/cloud-deployment/TenantIngestionModel.md` | names the slice-budget knob for a first ingest and its lease trade-off | — | doc read |

Wire note: additive only; no config leaf, no schema version, no migration — an existing checkpoint without the markers behaves exactly as today.

## Acceptance Criteria

- [ ] **AC-1** A repo whose slice ended `partial-progress` with `errors=0` is attempted on the next sweep (`sweepCadenceMs`), not after `effectiveCadenceMs`; a repo whose slice failed keeps today's backoff. Both arms as unit witnesses in `TenantRepoSyncService.spec.mjs`.
- [ ] **AC-2** Heavy-maintenance waiters still acquire the lease while a clean partial repo is re-admitted every sweep, including during a first ingest. Once a first slice has landed clean, `isBootstrapCriticalTask` ranks the lane ordinary, so a waiter starved past `fairnessYieldAfterMs` takes the lease on the ledger's clock; an unlanded or failed first slice keeps the class, and priority-zero still wins. Witness: a unit arm through the real classifier, picker and admission rule, with a checkpointed partial repo as the control. After deploy, the live `heavyMaintenanceStarvation` read shows `leaseHolder` moving off `tenant-repo-sync` (post-merge). The #415 receipt fields do not regress.
- [ ] **AC-3** *(deployed plane, `[L4-deferred — operator handoff needed]`)* on `neo-local-canonical`, the corpus tenant's `lastIngestedRev` is set and `corpusOutstanding.state: complete` within two days of the deploy that carries lever 1 alone (each slice still rebuilds the envelope, ~130 embeddings per ~6-minute slice against ~0.6 chunks/s measured throughput; lever 2 in #432 tightens this to one day); recorded on #64 AC-6 item 2 with the snapshot timestamp. Residual-Owner: #64.

Lever 2 — two consecutive clean partial slices on an unchanged mirror head do not re-materialize the envelope — was **split out 2026-09-23 12:5xZ to #432** (its own seam, `tenantRepoIngestEnvelopeBuilder`, its own witness); this ticket is lever 1 alone, and its ACs were renumbered 1–3 at the split.

## Out of Scope

- Embedding throughput itself (provider, model, host capacity — operator-owned).
- The blobless mirror's first-materialization round trips (#65).
- Over-ceiling chunks and their retry family (neomjs/neo#16972, #17336).
- Checkpoint contract revalidation (`tenantRepoCheckpointValidity.mjs`); the derived `checkpointStatus: failed` label for an attempted-but-unchecked repo reads wrong to an operator and may deserve its own name, but it is a projection, not this lane.

## Avoided Traps

- **Raising `sliceBudgetMs` globally.** It is the fairness bound that fixed sibling starvation (`configBase.mjs:2313`); a larger default moves that guarantee for every repo to serve one first ingest.
- **Reading the KB collection count as progress.** It moves only at slice boundaries; `corpusOutstanding` and the `embeddings=` log field are the instruments.
- **One blended ETA.** Rate depends on chunk size and provider; the AC binds a bound against the measured throughput, not a promise.
- **Wiring the skip into the yield primitive.** The safe-point budget is contract, not implementation detail (`configBase.mjs:2320`); the fix lives in the scheduler and the envelope path.

## Related

#64 (parent; AC-6 item 2 is the receipt this unblocks) · #411 / PR #424 (activation) · #65 · #25 · #415 / PR #418 · neomjs/neo#17416 (cornerstone 1) · `learn/agentos/cloud-deployment/TenantIngestionModel.md` · defect-note broadcast 2026-09-23 12:27Z

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T12:29Z (created-descending) — none equivalent; nearest #64 (parent), #65 (mirror round trips), #25 (yield wiring). A2A in-flight sweep (30 most recent, all read-states, 12:29Z): no claim on this shape — @neo-opus-ada #425/#428 (volumes), @neo-gpt engine lanes. MC sweep: `query_raw_memories` (6 results) — the 2026-08-18 klarso-plane measurement (~40 embeddings per sweep, apps-global ≈13 h) and the 2026-08-19 over-ceiling-chunk cause; no prior decision on re-admission or envelope reuse. Own-assignment sweep: #417, #23, #64, #65 — #64 is the parent, none equivalent. Structure map (`npm run ai:structure-map -- --files --loc`, 12:3xZ): owning folder `ai/daemons/orchestrator/services/` (44 files; `TenantRepoSyncService.mjs` at 2,103 code lines is already the folder's heaviest module — the scheduler change belongs in its `isRepoDue`/`partial-progress` seam, the envelope reuse in `tenantRepoIngestEnvelopeBuilder.mjs`, not as new lines on the service); sibling precedent #415 / PR #418 (the starvation receipt).

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("tenant repo sync partial-progress first ingest slice budget re-materialize envelope cadence corpus tenant embeddings per cycle")`

Authored by Vega (Fable 5.1, Claude Code) 🌿



## Timeline

- 2026-09-23T12:33:26Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T12:33:28Z @neo-opus-vega added the `bug` label
- 2026-09-23T12:33:28Z @neo-opus-vega added the `ai` label
- 2026-09-23T12:33:29Z @neo-opus-vega added the `agent-os` label
- 2026-09-23T12:33:43Z @neo-opus-vega added parent issue #64
- 2026-09-23T12:37:43Z @neo-opus-vega cross-referenced by #411
- 2026-09-23T12:37:43Z @neo-opus-vega cross-referenced by #64
- 2026-09-23T12:52:42Z @neo-opus-vega cross-referenced by #432
- 2026-09-23T12:57:22Z @neo-opus-vega cross-referenced by PR #433
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434
- 2026-09-23T14:17:15Z @neo-opus-vega cross-referenced by #438
- 2026-09-23T14:26:12Z @neo-opus-vega cross-referenced by PR #439
- 2026-09-23T14:39:40Z @neo-opus-vega cross-referenced by #440
- 2026-09-23T14:46:24Z @neo-opus-vega cross-referenced by #442
- 2026-09-23T14:58:15Z @neo-opus-vega referenced in commit `833187e` - "fix(tenant-sync): a first ingest ranks ordinary once its first slice has landed clean (#430)

A clean partial slice is due at the next sweep, but a first ingest keeps a
null lastIngestedRev until the whole corpus has landed, so
isBootstrapCriticalTask kept ranking it bootstrap-critical and the waiter
ledger never lets a lower rank displace it. Back-to-back slices would have held
the heavy lease for the entire first ingest while ordinary maintenance starved
past any bound.

The bootstrap class now buys the first slice only: a null-checkpoint entry
carrying a clean partial-resume marker ranks ordinary, so same-rank fairness
applies to the catch-up. An unlanded or failed first slice keeps the class,
and priority-zero tasks still win either way."
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444

