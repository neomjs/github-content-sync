---
id: 238
title: 'The tests own the sample roster and activity: a driver lands them, no spec reads the app''s seed'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-26T09:24:10Z'
updatedAt: '2026-09-26T13:35:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/238'
author: neo-fable-clio
commentsCount: 0
parentIssue: 237
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T13:35:40Z'
---
# The tests own the sample roster and activity: a driver lands them, no spec reads the app's seed

## Context

First leaf of #237 (the cockpit ships no sample data). The operator, 2026-09-26: "tests can use dummy data, but it should be part of tests, not the app." Before the app's seed can retire (the second leaf), every spec that boots on it must land its own fixture — so this leaf moves the data into the test tree and re-points the specs while the app still seeds, and lands on green suites.

## The Problem

Roughly thirty specs read the sample without saying so: the e2e Neural Link specs drag, select and count the eleven sample cards (`FleetCockpitDrillNL`, `FleetCockpitDrillRoundTripNL`, `FleetCockpitNWindowNL`, `FleetFirstLaunchNL`, `FleetGridScaleNL`, `FleetCardLifecycleNL`, `FleetGridKeyboardA11y`, `FleetPermanenceMatrixRow4NL`, `AgentCardSynthesisRenderNL`, `FleetNavFamilyPin`, `FleetCockpitBarCompositionNL`, `Cockpit`), the visual suite's `bootSettledCockpit` captures them, the component spec `RosterRefillSeam` refills over them, and the unit specs (`rosterStore`, `activityFeed`, `spineBanner`, `spineBannerPipeline`, `adapterWitness`, `preload`, `projection`, `cockpitFakes`, …) import `apps/agentos/config/fleetSampleData.mjs` or assert the `sample` state. When the seed leaves the app, all of them go red at once unless the fixture is theirs first.

## The Architectural Reality

- `apps/agentos/config/fleetSampleData.mjs` (`sampleActivity`, six events) and `apps/agentos/resources/data/fleetRoster.json` (eleven agents) — the data to move, verbatim, so no golden moves in this leaf.
- `test/playwright/visual/goldenPathEnvelope.driver.mjs` — the precedent: a module `Neo.worker.App.loadModule` imports INTO the App worker, which lands a fixture into the cockpit provider's stores the way the real read does; the e2e specs call `page.evaluate(path => Neo.worker.App.loadModule({path}))` with a `t` parameter per call.
- The cockpit's stores: `stores.fleetRoster` (`AgentOS.store.FleetRoster`, today `autoLoad: true` on the JSON url) and `stores.fleetActivityEvents`; the liveness owner's `publishConnection('grid' | 'stream', …)` sets the adapter states the surfaces render.
- `test/playwright/fixtures.mjs` (the e2e page fixture) and the visual spec's `bootSettledCockpit` — the two boot helpers the fixture landing joins.

## The Fix

1. `test/playwright/fixtures/fleetSample.mjs`: the roster rows and the activity events, moved from the app's two files (the app keeps its copies until the second leaf; this module is the tests' copy).
2. `test/playwright/fixtures/fleetSample.driver.mjs`: lands both into the mounted cockpit's provider stores from inside the App worker and publishes the `live` adapter states for them (the same seams the liveness owner uses), `?t=<n>` per call.
3. A page-side helper `landFleetSample(page)` beside the e2e fixtures and inside the visual spec's boot, called by every spec in the list above before it touches a card or an event; unit specs import the fixture module instead of the app's.
4. No spec references `apps/agentos/config/fleetSampleData.mjs` or `resources/data/fleetRoster.json` afterwards — a grep is the guard, cited in the PR.

## Acceptance Criteria

- [ ] AC-1 `test/playwright/fixtures/fleetSample.mjs` holds the eleven roster rows and six activity events byte-equal to the app's seed of this head; the app's files are unchanged in this leaf.
- [ ] AC-2 The driver lands the roster and the activity into the cockpit's stores and their surfaces read `live` with the fixture rows (an NL arm asserts eleven cards and six events after the landing on a cockpit booted without them — the arm boots with the app seed today, so it lands over it and asserts the fixture identities).
- [ ] AC-3 Every spec in the list lands the fixture explicitly (or imports the module); unit, component, e2e and visual suites green at the head; no golden moves.
- [ ] AC-4 `grep -rn "fleetSampleData\|fleetRoster.json" test/` returns nothing.

## Out of Scope

Retiring the app's seed and the `sample` vocabulary (the second leaf); the tasks pane (the third).

## Related

#237 (parent), #11 (the visual harness), #215 (the driver precedent).

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:22:32Z — no equivalent. A2A sweep: none. Memory Core sweep: none beyond the seed's JSDoc. Own-assignment sweep: #237 (the parent) only. Structure map: N/A.

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("fleetSample driver test fixtures land roster activity cockpit stores")`

## Timeline

- 2026-09-26T09:24:11Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-26T09:24:12Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T09:24:12Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T09:24:12Z @neo-fable-clio added the `ai` label
- 2026-09-26T09:24:12Z @neo-fable-clio added the `testing` label
- 2026-09-26T09:25:15Z @neo-fable-clio added parent issue #237
- 2026-09-26T10:04:50Z @neo-fable-clio cross-referenced by #249
- 2026-09-26T10:08:15Z @neo-fable-clio cross-referenced by PR #250
- 2026-09-26T10:20:23Z @neo-fable-clio referenced in commit `dd2428f` - "test(fleet): stamp the visual baselines for the re-captured synthesis goldens (#238)"
- 2026-09-26T11:53:39Z @neo-fable-clio referenced in commit `bc1da8d` - "merge(dev): fold dev into clio/238-test-fixtures, the baseline stamp regenerated over both golden sets (#238)"
- 2026-09-26T12:58:02Z @neo-fable-clio cross-referenced by PR #254
- 2026-09-26T13:00:36Z @neo-fable-clio referenced in commit `a3e8197` - "merge(dev): fold dev (the Observatory keeper view and engine pin 12) into clio/238-test-fixtures, the baseline stamp regenerated (#238)"
- 2026-09-26T13:00:36Z @neo-fable-clio referenced in commit `ddb8dd4` - "refactor(cockpit): the answered-surface admission is one path — AgentOS.util.FleetAdmission, called by the reads and by the tests' landing (#238)"
- 2026-09-26T13:04:10Z @tobiu referenced in commit `5c5d3c5` - "chore(visual): the baseline stamp follows the admission module into the style-owning inputs (#238)"
- 2026-09-26T13:35:40Z @tobiu referenced in commit `6386898` - "Merge pull request #250 from neomjs/clio/238-test-fixtures

test(fleet): the tests own the sample fleet — a landing seam in the App worker, no spec reads the app's seed (#238)"
- 2026-09-26T13:35:41Z @tobiu closed this issue

