---
id: 415
title: The starvation receipt names the lease holder but not why it let go
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-22T23:29:11Z'
updatedAt: '2026-09-23T01:50:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/415'
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
closedAt: '2026-09-23T01:50:28Z'
---
# The starvation receipt names the lease holder but not why it let go

## Context

Leaf of #64, its second open acceptance criterion (*"Expose the yield cause on the observation surface"*), split out so it can close on one PR. Live motivation tonight: @neo-gpt-emmy's read-only `healthcheck` at 2026-09-22T21:59Z showed `tenant-repo-sync` holding the heavy-maintenance lease while `core-corpus-projection`, `dream` and `graphlog-compaction` sat deferred — the receipt could say *that*, and nothing on the plane could say *why* the holder keeps or releases the lease. #411 (activating the `github-content-sync` tenant) will be diagnosed against exactly this surface.

## The Problem

`TenantRepoSyncService` records `leaseYielded` and `observedYieldCause` per cycle (`ai/daemons/orchestrator/services/TenantRepoSyncService.mjs:2646-2686`, summary at `:3366-3416`) and they reach the task's outcome details. The starvation watchdog's receipt (`ai/daemons/orchestrator/scheduling/heavyMaintenanceStarvationWatchdog.mjs:59-64`) carries `breaches[]` (`taskName`, `priorityZero`, `bootstrapCritical`, `deferredSince`, `starvedForMs`), `leaseHolder` and `leaseStatus` — never the holder's yield state. Memory Core's `HealthService.mjs:1157-1166` renders that receipt as `Heavy-maintenance starvation: <task> deferred since <t> (lease holder: <x>)`. So a reader sees the holder and the waiters and must infer, from two samples and a code read, whether the holder yielded, hit its slice budget, or simply re-acquired — #64 records that this inference cost two samples and an archaeology pass.

## The Architectural Reality

- The watchdog reads the scheduling ledger and the lease file, not task outcomes; the yield fields live in the holder task's last outcome (`state['tenant-repo-sync'].lastOutcome.details`), which the scheduler already persists for `inspect_deployment` (`scheduling/registry.mjs` reads `state['core-corpus-projection']` the same way at `:115`).
- The receipt is consumed read-only by Memory Core's health surface; adding fields is additive — `observation.breaches`/`observation.leaseHolder` are copied as-is.
- ADR 0022 locates fairness in release-time selection; this leaf adds no scheduling behaviour, only observation of the behaviour that exists.

## The Fix

1. The watchdog receipt gains `holderYield: {leaseYielded: Boolean|null, observedYieldCause: String|null, cycleAt: String|null}`, read from the current (or most recent) lease holder's last outcome details when that task records them (`tenant-repo-sync` does; other holders yield `null`s, never an absent key).
2. `TenantRepoSyncService`'s summary details carry `observedYieldCause` beside `leaseYielded` (`:3416`), so the value the log line prints is the value the receipt reads.
3. `HealthService`'s starvation detail appends `; holder yield: <cause|none observed>` and `observation.holderYield` carries the object.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| starvation receipt (`heavyMaintenanceStarvationWatchdog.mjs`) | the holder task's persisted last outcome | `holderYield` object, nulls when the holder records nothing | nulls, never a missing key | watchdog JSDoc | spec: holder with recorded yield → fields; holder without → nulls |
| `tenant-repo-sync` outcome details (`TenantRepoSyncService.mjs:3416`) | the cycle's own `observedYieldCause` | added beside `leaseYielded` | — | existing `@returns` | existing cycle-summary specs extended |
| MC `healthcheck` starvation detail (`HealthService.mjs:1166`) | the receipt | names the yield cause | `none observed` | handbook line | spec over a fixture receipt |

## Decision Record impact

`aligned-with` ADR 0022 (observation only; no preemption, no new hold bound). `none` amended.

## Acceptance Criteria

- [ ] **AC-1** — With a holder whose last outcome recorded `leaseYielded: true, observedYieldCause: '<cause>'`, the watchdog receipt carries `holderYield` with those values and `cycleAt`; with a holder that records nothing, it carries `{leaseYielded: null, observedYieldCause: null, cycleAt: null}` — asserted, not absent.
- [ ] **AC-2** — `TenantRepoSyncService`'s cycle summary details expose `observedYieldCause` beside `leaseYielded`; the existing summary specs assert both.
- [ ] **AC-3** — Memory Core's `healthcheck` starvation detail names the yield cause, and `observation.holderYield` carries the object; spec over a fixture receipt, positive and null arms.
- [ ] **AC-4** — #64's falsifier is preserved: no arm in this leaf asserts `leaseHolder: null` as health; the arms assert what the receipt SAYS about the holder, never that the lease is free.
- [ ] **AC-5** — *(plane, post-merge, recorded on #64)* one live `healthcheck` read on the container plane shows the field populated during a `tenant-repo-sync` hold.

## Out of Scope

- The fairness bound itself and the re-acquisition starvation (#64 AC-1; #17380 / #17398 history).
- `failed`-vs-`uninitialized` reporting (#64's other open AC; overlaps neomjs/neo#16551).

## Avoided Traps

- ⛔ Do not read the lease FILE for the cause — it holds owner and timestamps, never a yield decision; the cause lives in the holder's outcome.
- ⛔ Do not assert a free lease as health (#64's recorded falsifier: the holder was already `null` while three lanes starved).

## Related

#64 (parent) · #411 (the activation this surface will diagnose) · #237 · neomjs/neo#17380, #17398 (the yield predicate this observes) · ADR 0022

Live latest-open sweep: latest 20 open Brain issues at 2026-09-22T23:28Z — none equivalent (nearest #64 itself, #413, #411). A2A in-flight sweep (mailbox read continuously since 21:55Z): no claim on this surface; @neo-gpt-emmy's 21:59Z evidence is the trigger. Memory Core sweep: my 2026-08-19 work shipped the yield predicate (#17398) and recorded the gap this leaf closes; no prior decision against surfacing it. Own-assignment sweep: #402, #411, #362, #237, #64, #65, #23 — none equivalent. Structure map: N/A — existing files only, no new `.mjs`.

Origin Session ID: fc04c361-0cae-4a80-9506-fa2ef4785d2b
Retrieval Hint: "starvation receipt holderYield observedYieldCause leaseYielded healthcheck watchdog tenant-repo-sync"

## Timeline

- 2026-09-22T23:29:11Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-22T23:29:12Z @neo-opus-vega added the `enhancement` label
- 2026-09-22T23:29:12Z @neo-opus-vega added the `ai` label
- 2026-09-22T23:29:12Z @neo-opus-vega added the `agent-os` label
- 2026-09-22T23:29:21Z @neo-opus-vega added parent issue #64
- 2026-09-22T23:39:58Z @neo-opus-vega cross-referenced by #417
- 2026-09-23T00:04:51Z @neo-opus-vega cross-referenced by PR #418
- 2026-09-23T01:13:02Z @neo-opus-vega referenced in commit `61ebf60` - "feat(orchestrator): the holder's cycle time is the writer's own stamp — lastCompletionAt on every terminal mark (#415)"
- 2026-09-23T01:25:08Z @neo-opus-vega referenced in commit `9265233` - "chore(ci): merge dev into vega/415-holder-yield-cause — the executing list keeps both additions (#415)"
- 2026-09-23T01:41:10Z @neo-opus-vega cross-referenced by #420
- 2026-09-23T01:50:29Z @tobiu referenced in commit `0d9ad56` - "Merge pull request #418 from neomjs/vega/415-holder-yield-cause

feat(orchestrator): the starvation receipt names why the lease holder let go (#415)"
- 2026-09-23T01:50:29Z @tobiu closed this issue
- 2026-09-23T02:29:44Z @neo-opus-vega cross-referenced by #64

