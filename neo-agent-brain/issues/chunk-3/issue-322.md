---
id: 322
title: fleetTasks reduces the starvation receipts and the backup lane into task rows
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T23:32:05Z'
updatedAt: '2026-09-05T15:35:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/322'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-05T15:35:33Z'
---
# fleetTasks reduces the starvation receipts and the backup lane into task rows

## Context

The Tasks pane's producer (`ai/services/fleet/fleetTasksSource.mjs`) reduces the deployment-state snapshot's `tenantRepoSync`, `recoveryRuns` and `selfHeal` blocks into provenance-labeled rows and answers one `fleetTasks` envelope. The same snapshot has carried the heavy-maintenance starvation receipts since the watchdog lane (`ai/daemons/orchestrator/scheduling/heavyMaintenanceStarvationWatchdog.mjs`; `createDeploymentStateSnapshot` tolerates the block as additive) and the backup lane under `maintenance` — neither reaches the pane. The consumer design is neomjs/neo-agent-institution#113 (sub of its #10); this leaf is its producer half and follows that ticket's sketch approval. Peer input on the consumer ticket (Euclid, Emmy, 2026-09-04) fixed this leaf's contract: rows are `orchestrator`-sourced, the wait is text not progress, the lease is a summary fact, the cap is per section. Euclid's v2 review (2026-09-05) added the pre-cap total: the pane must be able to say *known · shown* across every queued producer under truncation.

## The Problem

On 2026-09-04 five heavy-maintenance tasks starved 2.7–11.8 h behind the `summary` lease while every container read *serving*; the cockpit could not show it. The receipts exist in the file the producer already reads.

## The Architectural Reality

The deployment reducer in `fleetTasksSource.mjs` (rows `{id, section, name, source, state, at, progress, detail}`; `makeProgress` accepts `determinate | backlog` and clamps `done` to `total` (`:97`); `MAX_ROWS = 12` applied per section (`:386`); `counts` are post-cap lengths; the fleet's redaction authority); the reader is `readDeploymentStateSnapshot` (`available` / `stale` carry the snapshot). The starvation block: `{posture, checkedAt, degradeAfterMs, waiterCount, unreadableCount, leaseHolder, leaseStatus, breaches[{taskName, priorityZero, bootstrapCritical, deferredSince, starvedForMs, leaseHolder, reasonCode, blockingTaskName, leaseOwner, leaseStatus}]}` — breaches exist only beyond `degradeAfterMs` (`:77`), two clocks and three causes are kept apart (`:69`), and the collector copies the persisted verdict verbatim into every snapshot (`DeploymentStateBridgeService.mjs:477`). The backup block: `maintenance.backup` / `lastBackup`. The consumer's `FleetTask` accepts `orchestrator` as a row source and rejects `deployment` (the envelope's axis key).

## The Fix

One more reducer in `fleetTasksSource.mjs`, and two additive summaries on the envelope:

- each breach → a `queued` row: `source: 'orchestrator'`, `name = taskName`, `state = 'starved'`, `at = deferredSince`, `progress = null` (a wait is not completion and the progress path clamps), `detail` = the row's own cause in words (`reasonCode`, `blockingTaskName`, `leaseOwner`) plus the `priorityZero` / `bootstrapCritical` flags; the waiting facts ride additive numeric fields on the row — `waitMs = starvedForMs`, `thresholdMs = degradeAfterMs`, `checkedAt` — so the pane renders unclamped text and, if the sketch earns it, a decorative marker;
- the envelope gains a `scheduler` summary beside the rows: `{leaseHolder, leaseStatus, checkedAt, degradeAfterMs, posture, starvedTotal, unreadableCount}` — `starvedTotal` is the breach count before the cap (never `waiterCount`, which includes below-threshold waiters); no acquisition time is invented for the holder;
- `counts` gains `queuedKnown` — the pre-cap total of every queued row the reducers produced across starvation, backup and tenant-sync producers; the post-cap number stays the existing `counts.queued`, so *known · shown* is `queuedKnown · queued` and the omission is their difference (no separate `shownCount`);
- the backup lane → a `queued` or `recent` row (`source: 'orchestrator'`) from `maintenance.backup.phase` and `lastBackup` (`unanchored` when no success is on record);
- the per-section cap stays; within `queued`, operationally blocked rows take display priority (deterministic, `starvedForMs` descending, then name) — display order, never scheduler order.

No new wire verb, no new leaf; names, enums, durations and timestamps only — no paths, no config.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleetTasks` queued rows from `heavyMaintenanceStarvation.breaches[]` | the watchdog receipt shape; the row contract | `source: 'orchestrator'`, `state: 'starved'`, `progress: null`, the row's own cause, `waitMs` / `thresholdMs` / `checkedAt` additive | block absent or posture `disabled` → no rows | producer JSDoc | red-first spec on the 09-04 plane fixture |
| envelope `scheduler` summary | `leaseHolder` / `leaseStatus` / `checkedAt` / `posture` + the pre-cap breach count | additive block beside the rows | block absent → summary absent, no claim | producer JSDoc | spec |
| envelope `counts.queuedKnown` | the queued rows before `MAX_ROWS` is applied, every producer | additive pre-cap total beside the post-cap `counts.queued` | no queued producer → `0`, equal to `counts.queued` | producer JSDoc | spec: more rows than `MAX_ROWS` → `queuedKnown − queued` = the omitted number |
| `fleetTasks` row from `maintenance` | the `DeploymentStateBridgeService` backup receipt | one backup-lane row, `orchestrator`-sourced | block absent → no row | producer JSDoc | spec |
| `MAX_ROWS` per section | the existing bound (`:386`) | blocked rows first, deterministic; omission readable from the counts | — | JSDoc | spec |

## Acceptance Criteria

- [ ] A fixture snapshot carrying the live plane VERBATIM (read 2026-09-05T12:49:36Z: three breaches behind `summary` — `dream`, `kbSync`, `message-concept-harvest`, 1.3–6.6 h — backup exhausted with no success on record, one tenant repo queued) reduces to three `orchestrator`-sourced queued rows with `state: 'starved'`, `progress: null`, unclamped `waitMs` and `thresholdMs`, the row's own cause fields as the wire sent them (this plane's writer predates `reasonCode`, so `null`, never invented — a second arm words a reason-coded breach), a `scheduler` summary with `starvedTotal: 3`, one backup row, and through the envelope `counts.queuedKnown: 6` beside `counts.queued: 6` (the REM digest backlog is the sixth queued producer). *(AC restated 2026-09-05 13:05Z from the 09-04 five-breach wording to the verbatim block the fixture carries.)*
- [ ] A snapshot without the blocks reduces to no such rows and no summary; posture `disabled` reduces to none; `counts.queuedKnown` then equals `counts.queued`.
- [ ] Control (a): two snapshots whose outer `generatedAt` advances while the inner `checkedAt` stays fixed reduce to identical rows, summary and counts.
- [ ] Control (b): a breach with no active holder, and an `unknown` posture with `unreadableCount > 0`, reduce to rows and a summary that say exactly that — no invented holder, no empty queue.
- [ ] Per-section cap: with more queued rows than `MAX_ROWS`, blocked rows come first in deterministic order and `counts.queuedKnown` exceeds `counts.queued` by the omitted number; `FleetTask` accepts every emitted row (`orchestrator` source).
- [ ] No path, config value or container detail crosses the wire — the existing leak arm extended with the new blocks.
- [ ] Implementation starts at neomjs/neo-agent-institution#113's sketch approval; the consumer lands against this head.

## Out of Scope

Acting on the lease; a new wire verb; the System view's `fleetDeploymentState` projection (#314 — keeps `posture` + `breachCount`); the scheduler's own decomposition (D#11857).

## Related

neomjs/neo-agent-institution#113 (consumer; peer input Euclid issuecomment-5547709648, Emmy issuecomment-5547752350; v2 review Euclid issuecomment-5548341320 — the pre-cap total, amended here 2026-09-05) · #314 / PR #315 · #64 (the orchestrator-side starvation lane) · neomjs/neo#17329 (the pane). Live latest-open sweep: latest 20 open Brain issues at 2026-09-04 ~23:35Z, no equivalent; A2A last-30 all-states at 23:35Z: no claim; Memory Core: session-scoped recall of the pane's origin session recovered the row contract; unscoped recall null, not clean.

Origin Session ID: 49133900-1f86-4134-a82b-30ff0709bcaf
Retrieval Hint: "fleetTasksSource heavy maintenance starvation breaches orchestrator source waitMs thresholdMs scheduler summary queuedKnown"


## Timeline

- 2026-09-04T23:32:05Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T23:32:07Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T23:32:07Z @neo-fable-clio added the `ai` label
- 2026-09-04T23:32:07Z @neo-fable-clio added the `agent-os` label
### @neo-fable-clio - 2026-09-04T23:32:32Z

Consumer half and the design gate: neomjs/neo-agent-institution#113 (sub of its #10) — this leaf's implementation starts at that ticket's AC-1 sketch approval.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49133900-1f86-4134-a82b-30ff0709bcaf

- 2026-09-04T23:32:33Z @neo-fable-clio cross-referenced by #113
- 2026-09-05T00:48:45Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-09-05T13:04:05Z @neo-fable-clio cross-referenced by PR #329
- 2026-09-05T13:38:30Z @neo-fable-clio cross-referenced by PR #117
- 2026-09-05T14:04:47Z @neo-fable-clio referenced in commit `8cc16b2` - "fix(fleet): the queue leads with blocked work before the cap and the backup lane keeps a null instant (#322)

orderSection ranks starved rows first (longest wait, then name) before any chronology, so the cap never cuts a waiter in favor of older ordinary rows. The backup lane is emitted whenever the plane observed it: a never-anchored lane or an unreachable receipt keeps its state and reason codes under the queue with a null instant instead of vanishing. Spec: the older-ordinary-row cap control, the null-instant arms, and the absent-maintenance / dated-success controls."
- 2026-09-05T15:35:34Z @tobiu closed this issue
- 2026-09-05T15:35:34Z @tobiu referenced in commit `55c07c0` - "Merge pull request #329 from neomjs/agent/322-tasks-starvation-rows

feat(fleet): the Tasks pane's queue carries the heavy-maintenance starvation receipt and the backup lane (#322)"

