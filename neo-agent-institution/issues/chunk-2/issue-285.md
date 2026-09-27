---
id: 285
title: Roster reconcile fires a load and a re-reconcile per departed resident
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - performance
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T11:49:30Z'
updatedAt: '2026-09-27T11:49:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/285'
author: neo-opus-grace
commentsCount: 0
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
---
# Roster reconcile fires a load and a re-reconcile per departed resident

## Context
Found while answering @neo-preview's review question on #283 (the review of `d5ccdf11`). #249's Fix asked for the departed residents to leave in "one batch, one `load`". #283 kept the per-id removal that was already there. Measured on dev `4ca542f` with a throwaway probe. It replays `FleetAdmission.admitRoster`'s reconcile half over a real `FleetRoster` with 20 residents, the fold armed and `rosterWired` set:

| Snapshot | `load` events | `reconcileRoster` calls |
|---|---|---|
| 20 → 6 (14 depart) | 27 | 15 |
| the same, one batched `store.remove(ids)` | 1 | 2 |
| 20 → 6 plus 1 joiner | 15 | 2 |
| the same, batched | 2 | 2 |

## The Problem
Two fan-outs compound:
1. `reconcileRoster` removes departures one id at a time. Every `store.remove(id)` is a `splice`, and every `splice` fires `mutate` → `load`, including for an id that only the unfiltered twin holds. The method's own comment already states the rule for adds: "one batched add — every store mutation fires `load`, per-row adds would fan out". The removal half breaks it.
2. `FleetAdmission.admitRoster` calls `reconcileRoster` outside the `reconcilingRoster` latch. Each `load` from the reconcile's own mutations enters `onRosterStoreLoad`, which runs `reconcileRoster(lastLiveRows)` again. Stopping that recursion is the latch's documented job. The first nested run does the remaining removals, and each later one is an idempotent pass over every snapshot row.

Every `load` reaches every roster listener: the grid, the fold census, the health counters. The work grows with departures × snapshot rows.

## The Architectural Reality
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs`: `reconcileRoster` (:589–609); `onRosterStoreLoad` holds the latch (:620–632); the latch's JSDoc (:96–104).
- `apps/agentos/util/FleetAdmission.mjs:91–92`: the unlatched call.
- Engine `src/collection/Base.mjs`: `remove(key)` is one `splice` for an array. `splice` fires `mutate` whenever it isn't silent, even when the view didn't change. `src/data/Store.mjs` `onCollectionMutate` turns that into a `load`. The `allItems` mirror carries an array's keys into the twin (neomjs/neo#18855).

## The Fix
- `reconcileRoster` removes all departures in one call: `store.remove(departed)`.
- The latch moves into `reconcileRoster`: set on entry, reset in `finally`. Every caller is covered, the admission included. `onRosterStoreLoad` keeps only its guard.

Expected: 14 departures → 1 `load` and 1 `reconcileRoster` call. With a joiner → 2 loads and 1 call.

## Contract Ledger Matrix
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The roster store's `load` events during a live reconcile | `LivenessController#reconcileRoster` | One `load` per batched mutation (at most two: joiners, departures) and one reconcile per admitted snapshot | None: a nested reconcile over the same snapshot is the defect | The method's and the latch's JSDoc | AC-1, AC-2 |

## Acceptance Criteria
- [ ] AC-1: A unit arm drives the real admission path (`loadRoster`, a bridge answer after live truth) over a real filtered `FleetRoster`. A 14-departure answer fires exactly one `load` and one `reconcileRoster` call. With one joiner, two loads and one call. Red on dev.
- [ ] AC-2: A `load` from another writer after live truth still re-applies the last snapshot (the source-precedence arms stay green), and #283's shrink arm stays green.
- [ ] AC-3: `FleetGridScaleNL`'s shrink arm passes.

## Out of Scope
- The engine firing `mutate` for a `splice` that changed nothing.
- The fold's threshold.

## Avoided Traps
- Suspending the store's events around the loop: listeners still need exactly one `load` for the change.
- Latching only in `admitRoster`: the latch stays a per-caller duty, and the next caller repeats the defect.

## Related
#249, #283, #237, neomjs/neo#18855

Decision Record impact: none.

Live latest-open sweep: latest 20 open issues read at 2026-09-27T11:47:52Z and again just before filing; no equivalent. Lexical `reconcileRoster` / `onRosterStoreLoad`: #249, #48, #98 (closed) and #42 (the view-layer debt matrix, not this). A2A claim sweep (30 newest, all read states): no claim on the reconcile. MC sweep ("reconcileRoster store.remove per resident load fan-out onRosterStoreLoad re-entry latch admitRoster"), 6 results: #249's filing context, no prior decision on batching. Own-assignment sweep: #278, #275, #11, none overlapping.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6
Retrieval Hint: "roster reconcile load fan-out per departed resident, latch outside the admission path"

Authored by Grace (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T11:49:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T11:49:32Z @neo-opus-grace added the `bug` label
- 2026-09-27T11:49:32Z @neo-opus-grace added the `agent-os` label
- 2026-09-27T11:49:32Z @neo-opus-grace added the `ai` label
- 2026-09-27T11:49:32Z @neo-opus-grace added the `performance` label
- 2026-09-27T11:49:57Z @neo-opus-grace cross-referenced by PR #283
- 2026-09-27T11:56:23Z @neo-opus-grace cross-referenced by PR #286
- 2026-09-27T12:09:01Z @tobiu referenced in commit `b5fbceb` - "fix(agentos): the live roster reconcile lands a snapshot as one batched add, one batched removal and one reconcile (#285)

reconcileRoster removed departures one id at a time, and every splice fires
a load. admitRoster reconciled outside the reconcilingRoster latch, so each
of those loads re-entered onRosterStoreLoad and ran the reconcile again over
the same snapshot: 27 loads and 15 reconciles for a 14-resident shrink.
Departures now leave in one store.remove, and the latch lives in
reconcileRoster itself, so every caller is covered."
- 2026-09-27T12:26:31Z @neo-opus-grace cross-referenced by #288

