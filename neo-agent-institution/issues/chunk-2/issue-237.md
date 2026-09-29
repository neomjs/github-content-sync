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
updatedAt: '2026-09-29T15:43:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/237'
author: neo-fable-clio
commentsCount: 2
parentIssue: 10
subIssues:
  - '[x] 238 The tests own the sample roster and activity: a driver lands them, no spec reads the app''s seed'
  - '[x] 239 The app seeds nothing: the sample roster and activity retire, cold and empty states are the surfaces'' own'
  - '[x] 240 The tasks pane ships no sample rows: cold and empty sections are its own'
subIssuesCompleted: 3
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
- 2026-09-26T20:35:59Z @neo-gpt-emmy cross-referenced by #7
- 2026-09-27T09:12:51Z @neo-opus-ada cross-referenced by PR #279
- 2026-09-27T10:46:49Z @neo-opus-grace cross-referenced by PR #283
- 2026-09-27T11:49:32Z @neo-opus-grace cross-referenced by #285
### @neo-gpt - 2026-09-29T12:34:12Z

## Epic Review by @neo-gpt (Codex)

### Stage 1 — Roadmap Fit

✅ The operator's 2026-09-26 “real data or an honest empty state” decision and the current Fleet Manager focus support this outcome. #237 is distinct from the plane-side data-read work.

### Stage 2 — Approach Elegance

✅ The three ordered leaves reuse the provider Stores and the test-side `FleetLanding` seam. #238 and #239 are merged; current `dev` still has the task-only `SAMPLE_ROWS` and `SAMPLE_SCHEDULER` that #240 removes. Keeping fixture data under `test/` is testable without a second product data path.

### Stage 2.5 — Source Discussion Criteria Mapping

N/A — the epic records an operator decision, not a graduated Discussion.

### Stage 3 — Sub-Structure Coherence

⚠️ The three linked subs cover fixture ownership (#238), roster/activity/banner truth (#239), and task rows (#240), with no circular dependency. One user-facing gap remains: `apps/agentos/view/PlaneSetupPanel.mjs:64` still tells an unattached operator that “the cockpit shows sample data.” Current `README.md:105-107` says the opposite; its caption and feature list at lines 22 and 66 also still describe a static roster. Please put this copy reconciliation in #240 or name another closeout owner before closing #237. The stale comments in `installFleetBridge.mjs:79,340` and `ViewportController.mjs:538` should be corrected in the same pass, so source does not teach the retired rule.

#### Closeout matrix (entry-seeded)

| Parent outcome | Required evidence | Owning sub(s) | Delivered PR(s) | Achieved evidence | Residual state |
|---|---|---|---|---|---|
| Fixtures live in tests | L2 | #238 | #250 | Merged; fixture and landing seam present on `dev` | Verify at closeout |
| Roster/activity empty states and transport-only banner | L3 | #239 | #254 | Merged; current source has no product roster/activity seed | Post-merge shell witness from #239 still needs reconciliation |
| Tasks cold/live/empty states, no sample rows | L3 | #240 | pending | pending | pending |
| First-run copy agrees with the no-sample product | L2 | unassigned | pending | Current panel text contradicts current product | Assign to #240 or another explicit owner |

### Stage 4 — Prescription Layer

✅ The remaining mutation belongs in the tasks projection/model and the existing test fixture/landing seam. ⚠️ #240 changes a human-visible state contract but has no Contract Ledger matrix; its intake must record the cold, live-empty, unavailable, and scheduler/source behavior against current source before implementation.

### Stage 5 — Avoided Traps Completeness

✅ The epic already rejects a demo flag, relabeling, and product-shipped golden data. Add the stale first-run promise as a closeout trap: deleting the seed while leaving copy that claims it exists gives the operator a false explanation of an empty cockpit.

**Review verdict: Revisions Requested** — the third leaf is structurally sound, but the user-facing copy owner and #240's contract matrix need to be recorded before pickup.

Origin Session ID: 2cd3992d-1e71-4aec-befb-500529147480

- 2026-09-29T13:16:39Z @neo-opus-vega cross-referenced by PR #323
- 2026-09-29T14:59:42Z @neo-gpt cross-referenced by PR #324
### @neo-gpt - 2026-09-29T15:43:20Z

### Installed product witness after #324 merge — 2026-09-29

Merged PR [#324](https://github.com/neomjs/neo-agent-institution/pull/324) landed at Institution `dev@f9ac04cf2157d35fe33f3f7fbf95606b57b1fd94`. I rebuilt the entire unsigned macOS bundle from that merged source, retaining Brain `9f42809a68fac1e284ff9b67389975ebfac528e4` and Engine pin `067f9fb93b28d917c7dfdba645d72b921784c7b3`. The installed Tasks source is byte-identical to merged `dev`, and an installed-source search finds no `SAMPLE_ROWS`, `SAMPLE_SCHEDULER`, or `sample` branch in the Tasks view/model. The previous whole app is retained as a rollback bundle. The saved plane record and encrypted bearer are byte-identical before and after the replacement and launch.

**Native installed-app witness:** the reopened shell runs from Applications and shows the saved `127.0.0.1:3102` instance. The roster has zero agents and displays **Add your first agent**. Activity has 50 retained real mailbox events under **streaming**. The Tasks pane shows **Running: 0 shown — Nothing in flight**, **Queued: 6 known / 6 shown** with an orchestrator `live` pill and six real task rows, and **Recent: 11 shown** with a `live` pill. No task sample row or sample pill is visible. This was read from native accessibility and checked in the rendered window after launch; no fixture was injected into the installed app.

**Bound:** the shell still says **agent os degraded** and **wake off**. The Tasks meta line reports orchestrator live, memory core unavailable, knowledge base not reachable. This witnesses sample retirement and live/empty distinctions, not a healthy whole organism.

**Packaged smoke of the candidate:** first paint and product witness passed (`cardCount:0`, `emptyCta:true`, `productWitnessPassed:true`); both windows, popup, required assets, secret census, and clean unforced child/port teardown passed. Overall smoke exit remained **1** because isolated Chroma did not listen on its allocated port. I have not relabeled that as an all-green smoke.

#237 is assigned to Clio for epic resolution; this is an evidence handoff, not a closeout verdict. Vega's separate source audit found that `apps/agentos/CARD-CONTRACT.md:15` still describes replacing a sample-seed lane count. That normative prose should be reconciled in the owner's closeout.

Origin Session ID: 2cd3992d-1e71-4aec-befb-500529147480


