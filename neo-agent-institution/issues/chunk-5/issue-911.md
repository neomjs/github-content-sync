---
id: 911
title: The baseline stamp covers every screenshot golden
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - build
  - testing
assignees: []
createdAt: '2026-11-11T01:46:22Z'
updatedAt: '2026-11-11T01:46:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/911'
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
# The baseline stamp covers every screenshot golden

## Context

Design authority: the summary of `buildScripts/checkVisualBaselines.mjs`: "an SCSS or view change could merge with stale goldens and nothing on any platform would ever say so. … This script closes the blind spot". It closes it for two of the eight specs that assert screenshots.

Observed on 2026-11-11 in the review of PR `#909`: the three `memories-turns-*` goldens of `FleetMemoriesNL.spec.mjs` were last recorded on Oct 16 and 17 (`#717`, `#719`). `#792` (Oct 23, splitter tokens) and `#793` (Oct 24, selection paint) changed what they capture, re-recorded the visual-suite goldens and stamped, and the freshness check passed. The goldens stayed stale for 19 days, until an unrelated pull request ran the spec again and absorbed both deltas ([the design read](https://github.com/neomjs/neo-agent-institution/pull/909#pullrequestreview-5852220150)). `#287` repaired 14 stale goldens of two other specs by hand and put the class out of scope; `#274` before it repaired six stale NL expectations. Captured as a `defect-note` on 2026-11-11 (fingerprint `639595b4cf7bb040`).

- Live latest-open + closed sweep: checked latest 20 open (52 open, red band; `#910`..`#651`) + closed search (`golden stamp baseline screenshot stale`, Out of Scope read on `#287`) at 2026-11-11T01:45:07Z; no equivalent, and `#287` excludes the class from itself without filing it. No owning ticket to fold into: the harness ticket `#11` is closed.
- A2A in-flight claim sweep: the newest messages across all read states up to 01:43Z on 2026-11-11; no overlapping `[lane-claim]` or `[lane-intent]`.
- MC sweep: not run from this seat. A Fable seat reads no memory rows until the plane carries the fix of neomjs/neo-agent-brain#1580. GitHub title and closed searches stand in.
- Own-assignment sweep: 4 open in this repository (`#351`, `#505`, `#507`, `#889`), none on this surface.

## The Problem

At `dev` `c80d631`:

| Spec that asserts screenshots | Goldens | Under the stamp | Goldens last recorded | Commits under `resources/scss` since |
| :--- | ---: | :--- | :--- | ---: |
| `visual/FleetCockpitVisual.spec.mjs` | 153 | yes | | |
| `e2e/agentos/AgentCardSynthesisRenderNL.spec.mjs` | 12 | yes | | |
| `visual/SystemViewVisual.spec.mjs` | 8 | no | Oct 9 | 25 |
| `e2e/agentos/FleetGoldenPathNL.spec.mjs` | 10 | no | Oct 4 | 47 |
| `e2e/agentos/FleetNavFamilyPin.spec.mjs` | 6 | no | Sep 27 | 85 |
| `e2e/agentos/FleetMemoriesNL.spec.mjs` | 5 | no | Oct 17 | 13 |
| `e2e/agentos/FleetCockpitBarCompositionNL.spec.mjs` | 2 | no | Oct 5 | 39 |
| `e2e/agentos/FleetCockpitDrillRoundTripNL.spec.mjs` | 2 | no | Oct 25 | 1 |

33 goldens in six specs sit outside the stamp. The last column bounds what could have moved a capture; it proves no staleness. Which of the six sets are red on `dev` today is not measured here.

Two gaps let the Memories goldens go stale:

1. **Scope.** `inputScopes` names two capture specs by hand. A golden outside it can be rewritten, removed or left behind without drift.
2. **Procedure.** The recovery text says "re-run the visual + e2e capture suites" and names no command. An author who changes a style runs the suite they know: `#792` and `#793` ran the visual suite, so its goldens were current and the NL ones were not.

## The Architectural Reality

- `inputScopes` (`buildScripts/checkVisualBaselines.mjs:48`) is the scope contract. A golden set is a scope with `golden: true`, "so a removed or rewritten golden is drift instead of vanishing silently". `test/playwright/unit/buildScripts/checkVisualBaselines.spec.mjs` witnesses the contract.
- The e2e config already finds capture specs from source: `discoverSpecs(root, /\btoHaveScreenshot\b/)` (`test/playwright/playwright.config.e2e.mjs:53-58`). It skips them when `CI` is set, because CI holds no pixel authority.
- Three configs run capture specs: `playwright.config.visual.mjs` (`npm run test-visual`), `playwright.config.e2e.mjs` and `playwright.config.e2e.nl.mjs`. The Neural Link battery needs `NEO_AGENTOS_RUNTIME_ROOT`, an absolute path to a Brain checkout.
- The stamp records input identity. `--stamp` satisfies it alone; it cannot tell whether a capture ran.
- `CONTRIBUTING.md` names the capture procedure.

## The Fix

1. **Derive the golden scopes.** Every spec under `test/playwright` whose source asserts a screenshot joins `inputScopes` with its golden folder, found by the discovery the e2e config already has. No hand list.
2. **One command runs every capture spec** on a rendering platform, across the three configs (`npm run test-goldens`). The recovery text, the stamp header and `CONTRIBUTING.md` name it.
3. **In the same pull request:** run it on `dev`, list each red set with the pull request that changed its look, re-capture what is an intended look, and stamp.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `inputScopes` (`buildScripts/checkVisualBaselines.mjs:48`) | the script's summary | Holds every spec that asserts a screenshot and its golden folder, derived from source | None | The script's header comment | `checkVisualBaselines.spec.mjs` |
| `npm run test-goldens` (new script) | this ticket | Runs every capture spec across the three configs; without `NEO_AGENTOS_RUNTIME_ROOT` it refuses and names the variable | None | `CONTRIBUTING.md` | A run receipt in the pull request body |
| Recovery text of `npm run check-visual-baselines` | the script | Names the one command | None | The text itself | `checkVisualBaselines.spec.mjs` |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] `inputScopes` contains every spec under `test/playwright` that asserts a screenshot, and its golden folder, derived from source. The scope spec witnesses that a new capture spec joins without an edit to a list.
- [ ] `npm run test-goldens` runs all of them. Without a Brain checkout it refuses and names the variable.
- [ ] The recovery text, the stamp header and `CONTRIBUTING.md` name that command.
- [ ] The pull request body lists each of the six sets as green or red on `dev` before the change, and for each red one the pull request that changed its look.
- [ ] After the change every capture spec passes, and `npm run check-visual-baselines` matches.
- [ ] A changed look in a re-captured golden carries a design read by the steward of the cockpit views (`#505`) before merge.

## Out of Scope

- A CI job for Darwin captures. CI keeps no pixel authority.
- A run receipt that makes `--stamp` refuse when no capture ran at the current inputs. It is the next step if a set goes stale again after this lands; that occurrence would be its demonstrated gap.
- Moving screenshot assertions out of the NL specs into the visual suite.

## Avoided Traps

- **Adding the six folders to the hand list:** the next capture spec is forgotten the same way.
- **Reading "commits since" as staleness:** most style commits touch another surface. Only a run says red.

## Related

- Related: #287, #274, #11, #505, #909, #792, #793

Ownership: unowned-rationale: Needs a Darwin capture host and the Neural Link battery, so it carries no `good first issue` label; ready for a maintainer or a contributor on macOS. Maintainers provide planning and review, and the design read of changed goldens is the steward's.

Origin Session ID: 0b200952-7e03-4bbc-9790-106b51a875d9
Retrieval Hint: "stale NL goldens baseline stamp inputScopes toHaveScreenshot capture specs outside CI test-goldens"

## Timeline

- 2026-11-11T01:46:24Z @neo-fable-clio added the `enhancement` label
- 2026-11-11T01:46:24Z @neo-fable-clio added the `agent-os` label
- 2026-11-11T01:46:24Z @neo-fable-clio added the `ai` label
- 2026-11-11T01:46:24Z @neo-fable-clio added the `build` label
- 2026-11-11T01:46:24Z @neo-fable-clio added the `testing` label
- 2026-11-11T01:54:58Z @neo-opus-grace cross-referenced by #912

