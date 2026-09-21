---
id: 78
title: FleetGridScaleNL reads back an empty store after possessing it over the wire
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T09:29:14Z'
updatedAt: '2026-09-04T14:13:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/78'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T14:13:35Z'
---
# FleetGridScaleNL reads back an empty store after possessing it over the wire

Sub-issue of #10 (the cockpit's proof surface). Split out of #73 / PR #75 (the battery that widened the glob and found it) on the reviewer's ask: a stated red needs an owner that outlives the merge.

## Context

`test/playwright/e2e/agentos/FleetGridScaleNL.spec.mjs` is the density-evidence scale witness: a 20-agent fixture loaded into the REAL provider-hosted `AgentOS.store.FleetRoster` over the Neural Link, then the mounted grid asserted (the online tier as cards, the idle tier folded to a count, the benched tail, the health counters). It is red at PR #75's head: headless under the Brain root on 2026-09-02 09:28Z (`NEO_AGENTOS_RUNTIME_ROOT=<brain> npx playwright test -c test/playwright/playwright.config.e2e.mjs agentos/FleetGridScaleNL --workers=1`) → 1 failed at the first store assertion, while `FleetCockpitDockNL` passed 2/2 in the same run.

**Observed** (PR #75's disposition, re-measured today): after `callMethod(roster.id, 'clear')` and `callMethod(roster.id, 'add', [fixture])`, `inspectStore(roster.id, 25)` reports `count` 0 (expected 20); the `add` call returns 20 `null`s. The sibling `AgentCardSynthesisRenderNL` witness possesses a roster with the same clear-then-add idiom and renders its rows. **Inferred, not verified:** the two witnesses resolve the store differently enough that this one addresses an instance the grid does not render from — the spec takes `findInstances({className: 'AgentOS.store.FleetRoster'})[0]`, and the registry has listed more than one FleetRoster-class store since the August rebuilds; 20 nulls from `add` reads like records the addressed store could not create.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-02T09:27Z — #73 (the parent battery leaf), #74 and #76 are adjacent, none owns this witness; A2A lane-claims in the last window: none on this scope.

## The Problem

A stated red with no owner past merge: PR #75 establishes the battery as the product's proof surface and names this witness red, but #73 closes with the PR. Until the cause is found, the scale contract (the mounted grid at the evidence's ceiling band) has no green anywhere, and the next Engine-pin bump or roster rebuild rots it unannounced.

## The Architectural Reality

- `test/playwright/e2e/agentos/FleetGridScaleNL.spec.mjs:64–79` — resolve the store by class, `clear`, `add(fixture)`, `inspectStore` (`count`, not `totalCount`, which is the remote-paging field).
- `apps/agentos/view/fleet/cockpit/Container.mjs` hosts the roster store in its `state.Provider` `stores` block (the projected grid binds `stores.fleetRoster`); `loadRoster` re-points it at the running fleet when the bridge wires up.
- `test/playwright/e2e/agentos/AgentCardSynthesisRenderNL.spec.mjs` — the sibling possession idiom that renders (compare how it picks the instance and when it writes).
- Neural Link surfaces: `findInstances`, `callMethod`, `inspectStore`, `getInstanceProperties`.

## The Fix

1. Reproduce with the instance question answered: list every `FleetRoster` instance (`findInstances` with `['id', 'count']`) and compare the id the grid binds (the grid's `store` through `getInstanceProperties`) with the one the spec addressed. If they differ, address the grid's store (the sibling's shape) — a witness repair.
2. If they match, the nulls are the defect: read what `add` does with the fixture rows (model field validation, `id` collisions with the seed, a `clear` that leaves the store loading), fix it in the store or the provider, and keep the witness as the falsifier.
3. Repoint the README battery paragraph from "cause open" to this ticket's outcome.

## Acceptance Criteria

- [ ] The instance question is answered in the PR (same store or not), with the `findInstances` receipt.
- [ ] `FleetGridScaleNL` green in its STORE half at the PR head — the possessed store answers 20 rows by poll on the twin, the view folds to the six named residents, title `Fleet · 20 agents`, chip `+14 idle · show` — with red-first stated against the unrepaired head (received 0). **Transferred to #98 (2026-09-04):** the DOM half (6 cards for 6 rows; 12 on the pinned engine) — its cause is neomjs/neo#18269, merged via neo PR #18273, and #98 is the engine pin that lands it; #98 AC-2 carries "green in full".
- [ ] The README's battery paragraph states this witness as a red with its cause and its owner named (#98) — never masked. **Transferred to #98 (2026-09-04):** the paragraph going red-free is #98 AC-4.

## Scope transfer (2026-09-04)

PR #95 delivers the consumer half on the pinned engine (the instance answer, the tier derived from `state`, the fold decided after the mutation settles, the corrected witness). The engine half — the unfiltered projection written inside the mutation and eager hydration before the splice — was filed from this measurement as neomjs/neo#18269 and merged via neo PR #18273 (`205bc52f8a`). The Institution consumes it through the engine pin #98, which owns the remaining two facts: `FleetGridScaleNL` green in full and a red-free README battery paragraph, plus retiring the microtask deferral this PR introduces.

## Out of Scope

- The other stated red (`AddAgentJourneyNL`, #74 / PR #77 — the engine's lazy rail item, neomjs/neo#18063).
- CI execution of Neural Link witnesses (#64).

## Related

Parent: #10. Origin: #73 / PR #75 (the disclosure and the reviewer's ask). Siblings: #74, #76.

Origin Session ID: 91f83b9c-df95-4f72-a68f-d33f470792ac

Retrieval Hint: `FleetGridScaleNL inspectStore count 0 add returns nulls store possession Neural Link`

📜 Clio


## Timeline

- 2026-09-02T09:29:16Z @neo-fable-clio added the `bug` label
- 2026-09-02T09:29:16Z @neo-fable-clio added the `ai` label
- 2026-09-02T09:29:16Z @neo-fable-clio added the `testing` label
- 2026-09-02T09:30:51Z @neo-fable-clio cross-referenced by #73
- 2026-09-02T09:38:10Z @neo-fable-clio cross-referenced by PR #75
- 2026-09-02T11:40:26Z @neo-fable-clio cross-referenced by PR #79
- 2026-09-02T14:03:54Z @neo-fable-clio cross-referenced by PR #77
- 2026-09-02T14:46:01Z @neo-fable-clio cross-referenced by #81
- 2026-09-02T14:52:33Z @neo-fable-clio cross-referenced by PR #82
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-02T15:47:41Z @neo-fable-clio cross-referenced by #84
- 2026-09-02T15:49:48Z @neo-fable-clio cross-referenced by #85
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T09:55:22Z @neo-fable-clio cross-referenced by PR #91
- 2026-09-04T10:08:33Z @neo-fable-clio cross-referenced by PR #93
- 2026-09-04T10:46:30Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T10:58:03Z @neo-fable-clio cross-referenced by #18269
- 2026-09-04T11:21:52Z @neo-fable-clio cross-referenced by PR #95
- 2026-09-04T11:46:05Z @neo-fable-clio referenced in commit `6345796` - "fix(agentos): the roster decides its idle fold after the mutation settles and derives the tier from state (#78)"
- 2026-09-04T12:01:27Z @neo-fable-clio cross-referenced by PR #18273
- 2026-09-04T13:02:26Z @neo-fable-clio referenced in commit `c569791` - "fix(agentos): the roster decides its idle fold after the mutation settles and derives the tier from state (#78)"
- 2026-09-04T13:02:26Z @neo-fable-clio referenced in commit `a2350f2` - "test(agentos): restamp on the merged dock-header goldens (#78)"
- 2026-09-04T13:04:46Z @neo-fable-clio cross-referenced by #98
- 2026-09-04T13:28:15Z @neo-fable-clio referenced in commit `29a560c` - "fix(agentos): the roster decides its idle fold after the mutation settles and derives the tier from state (#78)"
- 2026-09-04T13:28:15Z @neo-fable-clio referenced in commit `800f23b` - "test(agentos): restamp on the merged custody-heal sources (#78)"
- 2026-09-04T13:38:00Z @neo-fable-clio cross-referenced by #99
- 2026-09-04T13:51:25Z @neo-fable-clio referenced in commit `729a5ff` - "docs(readme): the battery's stated red names its owner — the engine pin #98 carries the merged fix (#78)"
- 2026-09-04T14:13:35Z @tobiu referenced in commit `d68f46b` - "Merge pull request #95 from neomjs/agent/78-fleet-grid-scale-witness

fix(agentos): the roster decides its idle fold after the mutation settles and derives the tier from state (#78)"
- 2026-09-04T14:13:36Z @tobiu closed this issue
- 2026-09-04T14:23:43Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-04T14:51:41Z @neo-gpt-emmy cross-referenced by PR #101

