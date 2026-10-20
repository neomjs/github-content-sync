---
id: 769
title: 'reconcileRoster clears and defers all row-mapped controlReason kinds, not only cleanup'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T18:45:11Z'
updatedAt: '2026-10-20T18:45:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/769'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 732
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
# reconcileRoster clears and defers all row-mapped controlReason kinds, not only cleanup

## Context

Follow-up surfaced while reviewing `neomjs/neo-agent-institution#768` (`Resolves #749`, which extends `SourceHealth.mapFleetSessionHealth` so all six Fleet lifecycle states map onto the roster card's vocabulary).

Design authority: `apps/agentos/CARD-CONTRACT.md` (Row 2 / `#477`: *"Pending takes visual priority over a prior reason, so a new attempt never displays a stale failure"*) and `#742` (`PR #744`), which established that `LivenessController#reconcileRoster` must (a) preserve an active C2 round-trip `controlReason` (`rejected` / `timeout` / `unauthorized` / `superseded`) or an in-flight `pendingAction` over a snapshot row's mapped `controlReason`, and (b) clear a row-mapped `controlReason` to `null` once a later snapshot row no longer carries one.

## The Problem

When `#742` (`PR #744`) landed, `SourceHealth.mapFleetSessionHealth` only ever returned a row-mapped `controlReason` with `kind === 'cleanup'`. Accordingly, `apps/agentos/view/fleet/cockpit/LivenessController.mjs:669–673` was written with a literal `kind === 'cleanup'` check:

```js
if (!next.controlReason && record.controlReason?.kind === 'cleanup') {
    next.controlReason = null
} else if (next.controlReason?.kind === 'cleanup' && (record.pendingAction || (record.controlReason && record.controlReason.kind !== 'cleanup'))) {
    delete next.controlReason
}
```

With `#749` (`PR #768`), `SourceHealth.mapFleetSessionHealth` (`apps/agentos/util/SourceHealth.mjs:279–302`) now produces row-mapped `controlReason` objects for five kinds (`cleanup`, `pending`, `failed`, `unavailable`, `exited`), and `RosterRow.mapRosterRow` (`apps/agentos/util/RosterRow.mjs:91`) spreads `...(sessionHealth.controlReason ? {controlReason: sessionHealth.controlReason} : {})` onto each mapped row — omitting `controlReason` when the seat is `running` or `stopped`.

Because `LivenessController#reconcileRoster` still checks only `kind === 'cleanup'`:
1. **Stale row-mapped reason after transition to `running` or `stopped`:** When a seat in `stopping` (`kind: 'pending'`), cleanup-free `failed` (`kind: 'failed'`), `unavailable` (`kind: 'unavailable'`), or `exited` (`kind: 'exited'`) transitions to `stopped` or `running` on a subsequent poll, `next.controlReason` is `undefined` and `record.controlReason.kind !== 'cleanup'`, so `record.set(next)` never clears `record.controlReason` to `null`. The card continues displaying `⚠ pending: stopping` or `⚠ failed: …` after the seat has settled.
2. **Round-trip C2 reason or in-flight `pendingAction` overwritten on re-poll:** While an operator-initiated Start or Stop on a `failed`, `unavailable`, `exited`, or `stopping` seat is in flight (`record.pendingAction`) or has settled with a C2 round-trip reason (`rejected` / `timeout` / `unauthorized` / `superseded`), a roster re-poll carrying `next.controlReason` with `kind` in `['pending', 'failed', 'unavailable', 'exited']` bypasses the `next.controlReason?.kind === 'cleanup'` guard and overwrites `record.controlReason`.

## The Architectural Reality

- Producer of row-mapped `controlReason`: `SourceHealth.mapFleetSessionHealth` (`apps/agentos/util/SourceHealth.mjs:279–302`) — emits `kind` in `['cleanup', 'pending', 'failed', 'unavailable', 'exited']`.
- Producer of C2 round-trip `controlReason`: `FleetLifecycleIntentAdapter` (`apps/agentos/util/FleetLifecycleIntentAdapter.mjs`) — emits `kind` in `['rejected', 'timeout', 'unauthorized', 'superseded']`.
- Reconciler: `LivenessController#reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:669–673`).

## The Fix

1. Export or define the frozen set of row-mapped `controlReason` kinds (`['cleanup', 'pending', 'failed', 'unavailable', 'exited']`, e.g. on `SourceHealth`) so `LivenessController#reconcileRoster` checks row-mapped membership rather than hardcoding `kind === 'cleanup'`:
   - If `!next.controlReason` and `record.controlReason?.kind` is a row-mapped kind, set `next.controlReason = null`.
   - If `next.controlReason?.kind` is a row-mapped kind and `(record.pendingAction || (record.controlReason && !isRowControlReasonKind(record.controlReason.kind)))`, `delete next.controlReason`.
2. Extend `test/playwright/unit/apps/agentos/view/fleet/cockpit/rosterStore.spec.mjs` (`#742` describe block) to cover `stopping` (`pending`), `failed`, `unavailable`, and `exited` transitions to `stopped`/`running` as well as round-trip C2 preservation across re-polls.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `LivenessController#reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:669–673`) | `apps/agentos/CARD-CONTRACT.md:21`; `#742`; `#749` | Clears any row-mapped `controlReason` (`cleanup`, `pending`, `failed`, `unavailable`, `exited`) when a later snapshot row carries none, and preserves active C2 round-trip `controlReason`s (`rejected`, `timeout`, `unauthorized`, `superseded`) or in-flight `pendingAction` over all row-mapped kinds | `null` when cleared | JSDoc on `reconcileRoster` | AC-1, AC-2 |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 `LivenessController#reconcileRoster` clears `record.controlReason` to `null` when a seat with any row-mapped `controlReason` kind (`cleanup`, `pending`, `failed`, `unavailable`, `exited`) transitions on a subsequent snapshot poll to a state without `controlReason` (`running` or `stopped`).
- [ ] AC-2 `LivenessController#reconcileRoster` preserves an active C2 round-trip `controlReason` (`rejected`, `timeout`, `unauthorized`, `superseded`) or an in-flight `pendingAction` over any snapshot row's mapped `controlReason` kind (`cleanup`, `pending`, `failed`, `unavailable`, `exited`).

## Out of Scope

- Changing the `SourceHealth.mapFleetSessionHealth` mapping or `FleetStartPlan` partitioning introduced in `#749` (`PR #768`).
- Visual baseline captures for `#749` (`AC-4` on `#749`, tracked under `#732`).

## Avoided Traps

- Checking individual `kind === '...'` strings inline in `LivenessController` without a shared row-kind predicate, which would drift again if another lifecycle word maps to a `controlReason`.
- Treating a newer row-mapped `controlReason` (e.g. `pending` from `stopping` transitioning to `failed` or `cleanup`) as blocked by the previous row-mapped `controlReason`: a row-mapped kind yields only to `pendingAction` or a non-row C2 round-trip `controlReason`, never to an older row-mapped kind.

## Related

- `neomjs/neo-agent-institution#749` / `neomjs/neo-agent-institution#768` (introduces the `pending`, `failed`, `unavailable`, and `exited` row-mapped `controlReason` kinds)
- `neomjs/neo-agent-institution#742` / `neomjs/neo-agent-institution#744` (original `cleanup` reconciliation guard)
- Part of `neomjs/neo-agent-institution#732`

## Sweep Attestations

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` at `2026-10-20T18:44:10Z` (`#763`, `#749`, `#732`, `#727`, `#687`, `#683`, `#505`, `#490`, `#485`, `#479`, `#477`, `#424`, `#414`, `#391`, `#351`, `#312`, …); no equivalent open ticket found.
- A2A in-flight claim sweep: checked latest 30 `status: 'all'` messages at `2026-10-20T18:44:14Z`; no competing claim on `LivenessController#reconcileRoster`.
- Memory Core rationale sweep: `query_raw_memories` on `reconcileRoster controlReason cleanup pending failed unavailable exited #742 #749` surfaced `#742`/`#744` (`45d491a8-f72d-4377-9ac9-f7db074f94a5`), confirming `#742` only guarded `kind === 'cleanup'` because `cleanup` was the sole row-mapped `controlReason` before `#749`.
- Own-assignment sweep: checked open issues assigned to `neo-gemini-julia` (`#763` only; no overlap).

Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85
Retrieval Hint: "LivenessController reconcileRoster controlReason cleanup pending failed unavailable exited #742 #749"

## Timeline

- 2026-10-20T18:45:11Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T18:45:12Z @neo-gemini-julia added the `bug` label
- 2026-10-20T18:45:13Z @neo-gemini-julia added the `agent-os` label
- 2026-10-20T18:45:13Z @neo-gemini-julia added the `ai` label
- 2026-10-20T18:45:13Z @neo-gemini-julia added the `testing` label
- 2026-10-20T18:46:14Z @neo-gemini-julia added parent issue #732
- 2026-10-20T18:52:21Z @neo-gemini-julia cross-referenced by PR #770

