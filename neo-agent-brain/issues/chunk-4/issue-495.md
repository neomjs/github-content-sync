---
id: 495
title: tenant-repo-sync re-ranks bootstrap-critical after every slice
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-25T15:43:35Z'
updatedAt: '2026-09-25T16:44:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/495'
author: neo-opus-ada
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
closedAt: '2026-09-25T16:44:31Z'
---
# tenant-repo-sync re-ranks bootstrap-critical after every slice

## Context

This is a leaf of #64, for AC-1: a due lane's wait after a holder releases is bounded.

Measured 2026-09-25 on the local plane, Brain `19be7e8`. `picker.mjs`, `pipeline.mjs`, `MaintenanceBackpressureService.mjs` and `tenantRepoSync.mjs` are identical there and at dev `94fe224`.

- Six heavy lanes have been deferred since 2026-09-24 18:42Z, each `heavy-maintenance-backpressure behind tenant-repo-sync`: dream, summary, memory-summary-backfill, message-concept-harvest, core-corpus-projection and graphlog-compaction. REM shows 990 undigested and 0 recent cycles.
- A new tenant slice starts 3 s after each `Cycle summary`.
- The ingesting repo `neo-shared/github-content-sync` has a numeric `partialProgressAt` and `consecutiveFailures: 0`. Under #430 the lane is therefore ordinary, not bootstrap-critical.
- The other four configured repos are ingested.
- The stale `neo-shared/neo` entry (5 failures, last attempt 2026-08-06) is not configured, since the sweep reports 5 repos. It is ignored.

Since 15:35Z the plane runs `94fe224` with the tenant lane paused by env, so the starved lanes can run.

## The Problem

`isBootstrapCriticalTask('tenant-repo-sync')` returns true without reading the manifest whenever the configured-coverage snapshot is older than its 60 s TTL. `refreshConfiguredTenantRepoLabels()` kicks the async refresh and reports it pending, and `if (coverageRefreshPending && Array.isArray(labels)) return true` fires before the manifest is read.

The snapshot is refreshed only when the tenant lane itself is evaluated, and a running lane never is:

- `filterAlreadyRunning` drops it before rank 1b.
- The waiters' own `isBootstrapCriticalTask` calls return false before the refresh line.

A slice (`sliceBudgetMs`, 5 min) outlasts the TTL, so the decision after every slice finds the snapshot expired and re-grants the class. #430's first-slice rule never takes effect in production. Its arm (`heavyMaintenanceWaiterLedger.spec.mjs` › 'a first ingest whose first slice landed clean ranks ordinary in the pick and at admission') stamps the snapshot fresh, and the real decision point never sees a fresh one.

Reproducer: a scratch probe against the real `pickNextCandidate` and service, using #430's clean-first-slice manifest with candidates `tenant-repo-sync` and `dream`.

| snapshot age | winner |
|---|---|
| 0 s | `dream` |
| 59 s | `dream` |
| 61 s | `tenant-repo-sync` |
| 303 s (after a slice) | `tenant-repo-sync` |

## The Architectural Reality

- `ai/daemons/orchestrator/services/MaintenanceBackpressureService.mjs` holds `CONFIGURED_TENANT_REPO_LABELS_TTL_MS` (60 s), `refreshConfiguredTenantRepoLabels` (the TTL check, which kicks `ensureConfiguredTenantRepoLabels`) and `isBootstrapCriticalTask` (the fail-safe that runs before the manifest read).
- `ai/daemons/orchestrator/scheduling/pipeline.mjs#runSchedulingPipeline` makes one pick per poll and binds `isBootstrapCriticalTask` into the picker's `policyContext`.
- `ai/daemons/orchestrator/scheduling/picker.mjs#selectByPriority`: rank 1b tests only surviving candidates, so a running lane is never tested.
- The fail-safe is deliberate and pinned by 'an expired coverage snapshot cannot dispatch REM while a canonical refresh discovers a new repo': a repo added to config since the last refresh must not lose its first decision to REM. The fix keeps that guarantee.

## The Fix

Keep the snapshot fresh on the poll cadence, not only when the lane is evaluated:

1. `MaintenanceBackpressureService`: `ensureConfiguredTenantRepoLabels` takes a maximum age, defaulting to the TTL. A new `warmConfiguredTenantRepoLabels()` refreshes ahead of the TTL, at half of it.
2. `runSchedulingPipeline` calls it every poll. The call is fire-and-forget, since the refresh never rejects.

While polls run, every decision then reads a snapshot inside its TTL. A clean first slice ranks ordinary (#430), and the most-stale waiter wins the decision after it. The fail-safe still covers a snapshot that is genuinely expired, such as at boot or with a stalled resolver.

## Acceptance Criteria

- [ ] AC-1: a red-first arm at pipeline level. A tenant slice longer than the TTL runs while polls continue, and the decision after it picks the starved waiter. The same sequence without the per-poll warm-up picks the tenant lane (the control).
- [ ] AC-2: `warmConfiguredTenantRepoLabels` refreshes a snapshot at least half a TTL old and leaves a younger one alone. Single-flight holds: one resolver call per refresh.
- [ ] AC-3: the fail-safe arm and the #430 arms pass unchanged.

## Post-Merge Validation

- [ ] After a recut that carries this, re-enable the tenant lane during a first ingest. A heavy waiter must run between tenant slices: the orchestrator log shows a waiter's run between two `Cycle summary` lines. Record it on #64.

## Out of Scope

- A starvation time bound on rank 1b. It is defense in depth for a genuinely bootstrap-critical lane that never lands a clean slice, and such a lane backs off on failure. #64 keeps the option.
- #25 (wiring the yield into tenant-repo-sync).
- The stale `neo-shared/neo` manifest entry. It is ignored while unconfigured, and only the unresolved-snapshot fallback reads it.

## Avoided Traps

- **Evaluating the expired snapshot instead of failing safe.** It fixes this case, but reverses the pinned guarantee that a new repo does not lose its first decision.
- **Raising the TTL above the slice budget.** That ties a cache lifetime to `sliceBudgetMs`, and a lease hold can run to `maxActiveHoldMs` (30 min).
- **Refreshing when a tenant run completes.** The next poll is about 3 s later and the refresh is async, so that is a race, not a guarantee.

## Related

#64 (parent, AC-1) · #430 and PR #433 (the first-slice rule this puts into effect) · #25

- Decision Record impact: none.
- Structure map: `ai/daemons/orchestrator/scheduling` and `ai/daemons/orchestrator/services`, existing files only.
- AiConfig: none touched.
- Live latest-open sweep: checked the latest 20 open Brain issues at 2026-09-25T15:40:43Z; no equivalent.
- A2A in-flight sweep: of the 30 most recent messages, the only claim on this scope is @neo-opus-vega's hand-off of #64 AC-1 (15:24Z) and her acceptance of this shape (15:39Z).
- MC sweep: "heavy-maintenance waiters starved behind tenant-repo-sync first ingest…" returned 5 results. The earlier mechanisms (fail-fast re-acquire, oversized chunks, unwired yield) are different ones, and none found this snapshot re-grant.
- Own-assignment sweep: none of my open Brain issues touches scheduling.

Origin Session ID: 0f80515e-7682-4313-8101-b926da48c55c
Retrieval Hint: "tenant-repo-sync bootstrap-critical re-granted by an expired coverage snapshot" · Brain dev `94fe224`

## Timeline

- 2026-09-25T15:43:35Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-25T15:43:37Z @neo-opus-ada added the `bug` label
- 2026-09-25T15:43:37Z @neo-opus-ada added the `ai` label
- 2026-09-25T15:43:37Z @neo-opus-ada added the `agent-os` label
- 2026-09-25T15:43:42Z @neo-opus-ada added parent issue #64
- 2026-09-25T15:48:46Z @neo-fable cross-referenced by #496
- 2026-09-25T15:57:04Z @neo-opus-ada cross-referenced by PR #498
- 2026-09-25T16:43:53Z @neo-opus-vega cross-referenced by #500
- 2026-09-25T16:44:31Z @tobiu referenced in commit `13f54e0` - "Merge pull request #498 from neomjs/ada/495-warm-coverage-snapshot

fix(orchestrator): a running tenant slice keeps the coverage snapshot warm (#495)"
- 2026-09-25T16:44:31Z @tobiu closed this issue

