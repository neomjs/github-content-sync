---
id: 224
title: 'A starved waiter reports no lease holder, and that word hides four different causes'
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-08-29T00:22:31Z'
updatedAt: '2026-08-29T10:00:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/224'
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
closedAt: '2026-08-29T10:00:48Z'
---
# A starved waiter reports no lease holder, and that word hides four different causes

Leaf of neomjs/neo-agent-brain#64 (epic). Implementation open as PR neomjs/neo-agent-brain#222.

## Problem

The heavy-maintenance starvation watchdog reports `leaseHolder` and nothing else about the lease. `pipeline.mjs` computes it as:

```js
const leaseHolder = inspection.active ? (inspection.lease?.owner ?? null) : null;
```

`inspectHeavyMaintenanceLeaseSync` returns `active: false` from **four** distinct statuses:

| status | meaning | `lease` |
|---|---|---|
| `missing` | `ENOENT` — no lease file at all | `null` |
| `stale` | file parsed, `isLeaseStale()` — **a holder died or expired without releasing** | the lease object |
| `unreadable` | read error | `null` |
| `malformed` | JSON parse failure | `null` |

**Three of the four mean a lease file exists in a bad state**, and `stale` is the one that explains waiters queued behind nobody. The discriminator is computed one line earlier and discarded.

## Live evidence

Measured from the deployment snapshot at 29s age, 2026-08-28T23:56:23Z:

```
posture      degraded        waiterCount 4        degradeAfterMs 3600000
leaseHolder  null

core-corpus-projection    deferred since 22:40:43Z   →   75 min
message-concept-harvest   deferred since 22:01:59Z   →  114 min
```

Both past the one-hour bound, and **unactionable as reported**. The watchdog's own WARN says *"the fairness yield bound has been exceeded; the lease pipeline is not admitting its waiters"* — but which of the four conditions produced it is unrecoverable from the record.

## Architectural reality

The watchdog is a **pure detector**: on `degraded` it records a `failed` health outcome, writes a WARN, persists the verdict, and calls `markCompleted`. Nothing preempts a holder or admits waiters. Detection is built; resolution is not, and that is #64 AC-1's territory.

**This must land before that fairness work is designed.** A fixture written today would encode whichever of the four causes its author guessed — and if the live cause is `stale`, the remedy is stale-lease recovery, not fairness scheduling. The two fixes have nothing in common beyond the symptom.

The snapshot projection spreads `...verdict`, so a field added to the verdict reaches `get_deployment_state_snapshot` with no further wiring.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| starvation verdict | `inspectHeavyMaintenanceLeaseSync`'s existing four-state `status` | carries `leaseStatus` alongside `leaseHolder` | a status the inspector does not emit cannot appear; no default invented | in-line rationale + projection JSDoc | assertions on both `missing` and `stale` |
| deployment-state projection | the existing verbatim `...verdict` spread | field reaches the snapshot with no new wiring | — | `collectHeavyMaintenanceStarvationSnapshot` JSDoc | spread is unchanged |

## Acceptance criteria

- [ ] A starvation verdict distinguishes *no lease file* from *a stale lease nobody recovered*, without a consumer re-reading the lease.
- [ ] The distinction survives into `get_deployment_state_snapshot`, since that is the consumed channel.
- [ ] A stale lease is asserted **together with** `leaseHolder: null` — it is the pair that shows why one field could not carry it.
- [ ] No new task, file, ledger or registry: one field on a record already persisted and projected.
- [ ] The existing healthy → degraded → cleared sequence test keeps its order-exact assertions; new cases append rather than splice.

## Out of scope

- **Acting on the posture.** Preemption, admission, or stale-lease recovery are #64 AC-1 and should be designed *after* this lands, on evidence rather than on a guess.
- **Changing `isLeaseStale` or the lease primitives.** This reads their existing verdict; it does not alter what stale means.
- **The four-state posture vocabulary** (`degraded`/`healthy`/`unknown`/`disabled`) — unchanged and correct.

## Avoided traps

- **Reading `leaseHolder: null` as "no holder, therefore a fairness problem."** It is one of four readings, and only one of them is a fairness problem.
- **Adding a diagnostic subsystem.** The reading already exists; it was being thrown away. One field, no machinery — the same bar #212 sets.
- **Filing before diagnosing.** I deliberately did **not** claim which cause is live tonight; that is what the field is for.

**Live latest-open sweep:** checked the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-08-29T00:20:47Z, created-descending; no equivalent found. Nearest neighbour is #211 (*"Routine Memory Core degradation triggers futile self-repair"*) — adjacent in spirit but a different subsystem (self-heal, not the heavy-maintenance lease) and not a duplicate. **A2A in-flight claim sweep:** 25 most recent messages scanned by recency and scope across the ~60-minute herd window; no overlapping claim.

Origin Session ID: 96836c41-0a29-415d-aa36-6ac60b81c782

Retrieval Hint: `query_raw_memories("heavy maintenance starvation leaseHolder null four causes missing stale unreadable malformed leaseStatus discriminator")`


## Timeline

- 2026-08-29T00:22:32Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-29T00:23:29Z @neo-opus-vega cross-referenced by PR #222
- 2026-08-29T00:27:26Z @neo-gpt-emmy added the `bug` label
- 2026-08-29T00:27:26Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T00:27:26Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T00:27:26Z @neo-gpt-emmy added the `architecture` label
- 2026-08-29T00:27:26Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T00:32:37Z @neo-opus-vega cross-referenced by PR #221
- 2026-08-29T10:00:48Z @tobiu closed this issue
- 2026-08-29T11:37:10Z @neo-opus-vega cross-referenced by #233
- 2026-08-29T16:52:22Z @neo-gpt cross-referenced by PR #234
- 2026-08-29T21:49:51Z @neo-opus-vega cross-referenced by #239
- 2026-08-29T22:18:59Z @tobiu cross-referenced by PR #242

