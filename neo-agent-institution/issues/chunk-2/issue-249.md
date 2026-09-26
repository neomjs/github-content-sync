---
id: 249
title: The live roster reconcile leaves folded idle residents behind when the fleet shrinks
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees: []
createdAt: '2026-09-26T10:04:49Z'
updatedAt: '2026-09-26T10:04:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/249'
author: neo-fable-clio
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
# The live roster reconcile leaves folded idle residents behind when the fleet shrinks

## Context

`AgentOS.view.fleet.cockpit.LivenessController#reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:591`) is the path every roster answer after the first takes: rows present in the snapshot are set in place, joiners are added, and "residents absent from the snapshot are removed". Found while landing the tests' fleet through the liveness owner's own admission (#238): `FleetGridScaleNL`'s shrink — a 20-resident fleet with 14 idle rows folded, then a 6-resident answer — leaves the fold chip reading "+14 idle · show". The replace path (`store.clear()` + `store.add()`) never showed it, because `clear` resets the unfiltered twin.

## The Problem

The removal census reads `store.items`, the filtered view. With the density fold active (`roster/Container.mjs` `foldThreshold_` 12) the idle residents live only in the store's unfiltered twin (`allItems`), so a snapshot that no longer carries them never removes them: the roster Controller's census reads `allItems` when a filter pass ran (`roster/Controller.mjs:85-100`), the fold keeps counting ghosts, and unfolding renders residents the fleet no longer has. A keyed `store.remove(agentId)` would not reach them either: `Neo.collection.Base#splice` (`node_modules/neo.mjs/src/collection/Base.mjs:1509`) removes only keys present in the filtered map and mirrors that (empty) removal into `allItems`. Live-only: the seed's replace path and the first landing are unaffected.

## The Architectural Reality

- `LivenessController#reconcileRoster(store, rows)` — the census and the removals.
- `roster/Controller.mjs` — the fold's census over `allItems` once a filter pass ran.
- `Neo.collection.Base#splice` — removal by key honours the filtered map; removing a filtered-out item needs the filters suspended, or the twin mutated and the filter re-run.
- `test/playwright/e2e/agentos/FleetGridScaleNL.spec.mjs` — the shrink arm carries a `test.fail` annotation naming this ticket (from #238); it flips to an unexpected pass when this lands.
- Structure map: N/A (app view layer).

## The Fix

Reconcile against the whole collection: census `(store.allItems ?? store).items` for the removals, and remove the departed residents through a path that reaches filtered-out records (suspend the roster's filters around the batch, or remove from `allItems` and re-filter) — one batch, one `load`. A unit arm over a filtered `FleetRoster` (idle rows folded, then a shorter snapshot) proves the twin shrinks; the NL shrink arm loses its annotation.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the roster store after a live snapshot | `LivenessController#reconcileRoster` | the store — filtered view and unfiltered twin — holds exactly the snapshot's residents | none: a missing removal is the defect | the method's JSDoc | AC-1, AC-2 |

## Decision Record impact

none.

## Acceptance Criteria

- [ ] AC-1 With the density fold active (more than 12 idle residents), a later snapshot without them removes them from the store's unfiltered twin; the fold count follows the snapshot (unit arm over a filtered store).
- [ ] AC-2 `FleetGridScaleNL`'s shrink arm drops its `test.fail` annotation and passes: 20 residents fold to "+14 idle", a 6-resident answer un-folds and titles "Fleet · 6 agents".

## Out of Scope

The replace path; the engine's keyed-remove semantics on a filtered collection (a Neo design choice — noted, not changed here).

## Related

#238 (found there), #237, the roster density fold (`roster/Container.mjs`).

unowned-rationale: a one-method fix beside #239's roster work — mine after #239 unless a peer takes it first.

Live latest-open sweep: the latest 20 open issues at 2026-09-26T10:00:50Z — no equivalent (#237–#247 are the sample-data epic and the shell / observatory lanes). A2A sweep: none. Memory Core sweep: not run — the mechanism is a source read, the three files cited above. Own-assignment sweep: none. Structure map: N/A.

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("reconcileRoster filtered store allItems fold shrink ghost idle residents")`

## Timeline

- 2026-09-26T10:04:50Z @neo-fable-clio added the `bug` label
- 2026-09-26T10:04:51Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T10:04:51Z @neo-fable-clio added the `ai` label
- 2026-09-26T10:08:15Z @neo-fable-clio cross-referenced by PR #250

