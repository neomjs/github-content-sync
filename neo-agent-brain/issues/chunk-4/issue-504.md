---
id: 504
title: 'A yield to a starving waiter abstains, and nothing dispatches the waiter'
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T17:54:38Z'
updatedAt: '2026-09-25T18:14:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/504'
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
closedAt: '2026-09-25T18:14:16Z'
---
# A yield to a starving waiter abstains, and nothing dispatches the waiter

## Context

Local plane, 2026-09-25, Brain dev `bc6ad07` (carries #498). The dream lane last ran 16:08–16:10Z and has been a heavy-maintenance waiter since 16:20:10Z (`heavy-maintenance-waiters/dream.json`: `reasonCode heavy-maintenance-backpressure`, refreshed every poll). Measured, from the orchestrator log and the lease file:

| Time | What the log shows |
|---|---|
| 17:38:20Z | tenant slice ends (`Cycle summary … 1 partial-progress`) |
| 17:38:23Z | `Deferring tenant repo sync (cloud); heavy maintenance task REM sleep graph extraction is active (periodic-sweep:60000; yielding to starving dream (deferred since 16:20:10Z))` |
| 17:39:07Z | `Refreshing neo-shared/github-content-sync` — the tenant lane re-acquired; no dream run in between |
| 17:44:09 / 17:44:11 / 17:45:05Z | the same three lines, 54 s apart again |
| 17:52:03Z | tenant lane paused (fragment `"false"`, orchestrator recreated); the next acquirer, `memory miniSummary backfill`, is deferred the same way: `yielding to starving dream` — and dream still does not run (`dream.running false`, `lastRunAt 16:08:07Z`, no new `rem-runs` receipt) |

So #498 holds: the tenant lane is ordinary at the post-slice decision (the yield fires, which the rank gate forbids for a bootstrap-critical acquirer). The yield itself is the defect: it makes the acquirer abstain, no one dispatches the waiter it yielded to, and the next poll re-selects the same acquirer. With the tenant lane off, every short-cadence heavy lane does the same thing in turn, so the dream has been starved since 16:20Z by lanes that each politely stepped aside for it.

## The Problem

Two defects, one outcome.

**1. The yield is admission-side and one-winner-per-poll makes it a pause, not a handoff.** `MaintenanceBackpressureService.acquireLeaseAndExecute` (`:985-1020`) runs `findWaiterToYieldTo`; when a waiter qualifies it records `heavy-maintenance-yield-to-waiter` and returns `false`. `runSchedulingPipeline` dispatches exactly one winner per poll (`pipeline.mjs:262-292`), so the poll ends with nothing running. The pipeline already states this about the bootstrap rank (`pipeline.mjs:270-274`): *"an admission-side yield makes the winner abstain without promoting the starved task, so the bootstrap lane would never be dispatched at all"* — which is why bootstrap is bound at selection. The fairness yield was left admission-side, and it has exactly the failure the comment predicts. `selectByPriority` (`picker.mjs`) never sees the waiter ledger: priority-0, then bootstrap, then staleness ratio — and a waiter on a long cadence (dream, 3,600,000 ms) loses the ratio to every short-cadence lane forever. I recorded the same shape on 2026-08-19 for #17380 ("a cooperative yield does nothing about a holder that returns promptly and comes straight back; that is an admission-side problem at the picker").

**2. After a yield the coverage snapshot goes cold, and the fail-safe re-grants rank 1b.** #498 warms `configuredTenantRepoLabels` only while `tenant-repo-sync` is `running`. A lane that has just yielded is not running, so nothing warms the snapshot; it expires within 60 s and `isBootstrapCriticalTask` fails safe (`refresh pending && labels array ⇒ true`), the lane outranks the waiter, `findWaiterToYieldTo` no longer qualifies the waiter (rank gate), and the lane acquires. That is the 54 s: the snapshot's remaining freshness. #498's delta note ("a slice is the only time the snapshot outlives its TTL unobserved") missed the yield window, and the yield window is the one that matters.

## The Architectural Reality

- Picker: `ai/daemons/orchestrator/scheduling/picker.mjs` `pickNextCandidate` (three filters, then `selectByPriority`: priority-0 → `isBootstrapCriticalTask` → staleness ratio → registry order). `policyContext` carries the bootstrap oracle; nothing carries the waiter ledger.
- Pipeline: `ai/daemons/orchestrator/scheduling/pipeline.mjs:237-292` — the warm-up gate (`:240-243`, keyed on `running`), `pickNextCandidate`, one `executeCandidate` for the winner; `context.enables.tenantRepoSync` (`:160`) and the registry (owned lanes only, `Orchestrator.mjs:1572`) are the ownership predicate the boot prewarm uses.
- Yield: `MaintenanceBackpressureService.mjs:985-1020` (`listActiveWaitersSync` → `findWaiterToYieldTo` → `recordDeferral` → `return false`); `heavyMaintenanceWaiterLedger.mjs:234-290` (`findWaiterToYieldTo`: a higher-ranked waiter always qualifies, same rank needs `fairnessYieldAfterMs` and an older `deferredSince`, a lower rank never).
- ADR 0022 §2.1: the lever is one-winner-per-poll, fairness acts at release-time selection, and hard preemption of a running holder is the anti-anchor. §2.4 AC-2: the return shape stays `{winner: Object|null}`. Its re-review trigger (line 128) applies to this ticket: a change to the picker's selection policy cites the ADR.

## The Fix

1. **Promote at selection.** In `runSchedulingPipeline`, after `pickNextCandidate`, evaluate the fairness decision for the would-be winner with the same inputs the admission path uses (`listActiveWaitersSync`, `findWaiterToYieldTo` with the winner's `priorityZero` / `bootstrapCritical` / `deferralStreakStartedAt`). If the waiter it would yield to is itself a surviving candidate (due, not running, no heavy conflict), dispatch the waiter as this poll's winner. The admission-side yield stays as the backstop for out-of-process acquirers (the manual scripts) and for a waiter that is not a candidate this poll. `{winner}` keeps its shape (ADR 0022 AC-2); the promoted waiter is the winner.
2. **Warm the snapshot while the lane is owned and enabled, not only while it runs.** The gate at `pipeline.mjs:240` becomes: the `tenant-repo-sync` descriptor is in the registry (owned) and `context.enables.tenantRepoSync` — the boot prewarm's own boundary — so a host edge or a disabled lane still never reaches the resolver, and an idle or yielding lane never re-earns rank 1b from an expired snapshot.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `runSchedulingPipeline` winner selection | ADR 0022 §2.1 (release-time selection); this ticket | a starving waiter the winner would yield to is dispatched instead of the abstaining winner | no ledger / unreadable ledger → today's selection, logged | red-first pipeline arm + control |
| `MaintenanceBackpressureService` (a read-only `findWaiterToPromote`-shaped seam exposing the yield decision to the pipeline) | this ticket | pure over the ledger read; no lease touch | absent seam → no promotion | unit arm |
| `acquireLeaseAndExecute` yield path | unchanged | backstop only | — | existing arms pass |
| warm-up gate (`pipeline.mjs:240`) | #498, amended | warms while the lane is owned and enabled | not owned / not enabled → never resolves coverage (#498's gate arm, re-keyed) | red-first arm: the decision 60 s after a yield still reads fresh coverage |

**Decision Record impact:** aligned-with ADR 0022 (the one-winner-per-poll lever applied where the ADR places fairness, release-time selection; no preemption of a running holder; `{winner}` shape preserved). Cited per the ADR's own re-review trigger; no §2.5 escalation path is opened.

## Acceptance Criteria

- [ ] AC-1: red-first pipeline arm. Two heavy candidates, the short-cadence one wins by staleness ratio, the long-cadence one is a ledger waiter older than `fairnessYieldAfterMs` at the same rank: the poll dispatches the waiter. Control: without the promotion the winner yields and the poll dispatches nothing.
- [ ] AC-2: the rank gate holds. A bootstrap-critical or priority-0 winner is not displaced by an ordinary waiter; a waiter that is not a candidate this poll (not due, running, or heavy-conflicting) is not dispatched, and the admission-side yield still records its deferral.
- [ ] AC-3: red-first arm for the warm-up. With the lane enabled and idle (not running) and the snapshot at 5 minutes old, the next poll refreshes it; with the lane disabled or unowned, the resolver is never called (#498's gate arm, re-keyed).
- [ ] AC-4 (post-merge, local plane, receipt on #64): with the tenant lane re-enabled during its first ingest, the dream runs between two tenant `Cycle summary` lines, and `get_rem_pipeline_state.recentCycles` gains the run.

## Out of Scope

- Removing the admission-side yield (it is the backstop for lease-aware scripts).
- Bounding rank 1b in time for a repo that cannot land a first slice (#65's case); on this plane `neo-shared/neo` is a stale manifest entry outside configured coverage and does not hold the class.
- The tenant lane's slice budget and cadence.
- The `undigested / digested` window semantics (#500, out of scope there too).

## Related

#64 (parent; this is AC-1's second half) · #495 / PR #498 (the TTL half; this amends its gate) · #430 (the first-slice rule, kept) · #17380 / PR #17397 (the 2026-08-19 record of the same shape) · #415 (the yield-cause observability that made this measurable) · ADR 0022 · #500 (the receipts this needs from mc-server)

Live latest-open sweep: the latest 20 open issues at 2026-09-25T17:51:08Z; no equivalent (#495 is the TTL half, closed by #498).
A2A in-flight sweep (all read states, last 60 min, read 17:56Z): claims on Institution #213, Brain #503, neo #19230; none on the picker or the yield.
MC sweep: "tenant lane yields to starving dream at slice end but dream never runs, the yield leaves the poll without a winner": my 2026-08-19 record of the shape (#17380), Ada's 2026-09-25 11:10Z mechanism reading (#64), and the pipeline's own bootstrap comment; no prior decision that promotes a waiter at selection.
Own-assignment sweep: 7 open; #64 is mine and this is its AC-1; none owns the picker change.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0, run 16:40Z): `ai/daemons/orchestrator/scheduling/` owns the picker and pipeline; `services/MaintenanceBackpressureService.mjs` the yield.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e
Retrieval Hint: "yield to starving waiter abstains one winner per poll promote waiter at selection dream starved coverage snapshot cold after yield"

## Timeline

- 2026-09-25T17:54:38Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T17:54:39Z @neo-opus-vega added the `bug` label
- 2026-09-25T17:54:40Z @neo-opus-vega added the `ai` label
- 2026-09-25T17:54:40Z @neo-opus-vega added the `architecture` label
- 2026-09-25T17:54:40Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T17:55:13Z @neo-opus-vega added parent issue #64
- 2026-09-25T18:00:22Z @neo-opus-ada cross-referenced by PR #498
- 2026-09-25T18:02:20Z @neo-opus-vega cross-referenced by #500
- 2026-09-25T18:02:33Z @neo-opus-vega cross-referenced by #64
- 2026-09-25T18:03:26Z @neo-opus-vega cross-referenced by PR #505
- 2026-09-25T18:14:16Z @tobiu referenced in commit `7d6a2cc` - "Merge pull request #505 from neomjs/agent/504-promote-starving-waiter

fix(orchestrator): the picker promotes the starving waiter it would yield to (#504)"
- 2026-09-25T18:14:16Z @tobiu closed this issue
- 2026-09-25T18:17:30Z @neo-opus-vega cross-referenced by #84

