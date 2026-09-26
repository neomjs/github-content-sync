---
id: 237
title: 'Epic: the cockpit ships no sample data — real data or an honest empty state'
state: OPEN
labels:
  - agent-os
  - ai
  - design
  - epic
assignees:
  - neo-fable-clio
createdAt: '2026-09-26T09:23:40Z'
updatedAt: '2026-09-26T09:23:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/237'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues:
  - '[x] 238 The tests own the sample roster and activity: a driver lands them, no spec reads the app''s seed'
  - '[x] 239 The app seeds nothing: the sample roster and activity retire, cold and empty states are the surfaces'' own'
  - '[ ] 240 The tasks pane ships no sample rows: cold and empty sections are its own'
subIssuesCompleted: 2
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Epic: the cockpit ships no sample data — real data or an honest empty state

Terminal predicate: the Fleet Manager cockpit carries no sample data — every surface renders what its read answered (live rows, an honest empty state, a cold "not answered yet", a stale or unavailable verdict), the banner says what the transport knows and never what data is showing, and the tests land their own fixtures.

## Problem scope

The cockpit's "honestly-labelled sample seeds" (`apps/agentos/config/fleetSampleData.mjs` with six seeded activity events, `apps/agentos/resources/data/fleetRoster.json` with eleven agents, the tasks pane's `SAMPLE_ROWS` / `SAMPLE_SCHEDULER`) were the zero-setup first paint of the pre-plane cockpit: a surface taught its shape before any bridge answered, and labelled itself `sample`. The operator's ruling of 2026-09-26, after the team shell's first plane-attach: *"removing the static roster and dummy activity items is crucial too. either there is real data, or there is not."* and *"tests can use dummy data, but it should be part of tests, not the app."* The witness that day: the shell attached to the plane, the Golden Path read was live, and the roster still showed eleven invented agents under a banner reading "fleet offline · showing the static roster" while the registry was simply empty — the sample hid the true state behind a plausible one.

Why an epic, not a ticket: the seed is consumed by three surfaces (roster, activity, tasks), by the banner vocabulary (`SpineBanner` reads `state === 'sample'`), by the rebinding path (`TargetBinding` republishes `sample`), by the harness's product witness (`adapterWitness.mjs`: `sample` in `ADAPTER_STATES`, `cardCount > 0`), and by roughly thirty specs and a dozen goldens that boot on the sample. Three one-PR leaves in a fixed order — fixtures first, so the app's removal lands on green suites — serve one outcome.

## Intended solution shape

1. **Tests own the fixtures.** The roster and activity data move verbatim into the test tree; a driver lands them into the cockpit's provider stores from inside the App worker (the `goldenPathEnvelope.driver.mjs` precedent), with a page-side helper for the e2e and visual suites; unit specs import the fixture module. No spec reads the app's seed after this.
2. **The app seeds nothing.** `fleetSampleData.mjs` and `fleetRoster.json` retire; the stores start empty; the adapter vocabulary loses `sample` (cold · live · stale · degraded). The surfaces' own empty states carry the message — the roster's bootstrap CTA "Add your first agent" (a design ruling on record), the activity stream's "no activity yet", the tasks pane's empty sections — and a cold surface says "not answered yet" in its own words. The banner keeps to transport verdicts. The smoke's product witness asserts the empty state or real cards, never a count of invented ones.
3. **Same rule for the tasks pane** (own pane, own goldens).

Design rule from here: no view or store seeds itself; a surface with nothing to show says so, in its own words, in the ink of its state.

## Out of scope

The plane-side composition of the roster / activity / tasks reads in plane-attach (the fleet child answering from the bundled organism instead of the plane — a Brain lane, defect-noted 2026-09-26); the Add-agent flow itself; the instance switcher's behaviour in the packaged shell (Ada's lane).

## Avoided traps / rejected shapes

- A "demo mode" that keeps the sample behind a flag — the ruling is binary, and a flag is a seed with a switch.
- Keeping the sample only for the visual goldens — the goldens capture fixtures the driver lands; a golden over app-shipped fake data would pin the thing being removed.
- Re-labelling the sample ("demo roster") — the label was already honest; the data was still invented.

## Related

#10 (the design-led surface, parent), #13 (design conformance), #11 (the visual harness the goldens run through), #212 / #228 (the plane-attach path that exposed it), neomjs/neo-agent-brain#83 and neomjs/neo-agent-brain#53 (the plane-side registry, where the real roster will come from).

Live latest-open sweep: the latest 20 open issues of this repository at 2026-09-26T09:22:32Z — no equivalent. A2A in-flight sweep 09:2xZ: no claim (Ada informed, her lane stays the switcher). Epic-layer sweep: the open epics' terminal predicates (#7, #8, #9, #10, #13, #24) — none names the sample data. Memory Core sweep: the seed's own JSDoc is the prior ruling ("the zero-setup first paint … labelled honestly"); this epic supersedes it on the operator's 2026-09-26 words. Structure map: N/A (`apps/agentos` and `test/playwright` siblings name the placement).

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("cockpit sample roster removed honest empty state fixtures in tests")`

## Timeline

- 2026-09-26T09:23:40Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-26T09:23:41Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T09:23:41Z @neo-fable-clio added the `ai` label
- 2026-09-26T09:23:41Z @neo-fable-clio added the `design` label
- 2026-09-26T09:23:41Z @neo-fable-clio added the `epic` label
- 2026-09-26T09:23:48Z @neo-fable-clio added parent issue #10
- 2026-09-26T09:24:11Z @neo-fable-clio cross-referenced by #238
- 2026-09-26T09:24:44Z @neo-fable-clio cross-referenced by #239
- 2026-09-26T09:24:58Z @neo-fable-clio cross-referenced by #240
- 2026-09-26T09:25:15Z @neo-fable-clio added sub-issue #238
- 2026-09-26T09:25:16Z @neo-fable-clio added sub-issue #239
- 2026-09-26T09:25:18Z @neo-fable-clio added sub-issue #240
- 2026-09-26T09:34:23Z @neo-opus-ada cross-referenced by #241
- 2026-09-26T09:34:29Z @neo-opus-ada cross-referenced by #244
- 2026-09-26T09:34:32Z @neo-opus-ada cross-referenced by #246
- 2026-09-26T09:34:33Z @neo-opus-ada cross-referenced by #247
- 2026-09-26T10:04:50Z @neo-fable-clio cross-referenced by #249
- 2026-09-26T12:17:42Z @neo-opus-grace cross-referenced by PR #250
- 2026-09-26T12:44:06Z @neo-fable-clio cross-referenced by PR #254

