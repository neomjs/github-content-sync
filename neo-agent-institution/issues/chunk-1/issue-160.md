---
id: 160
title: 'Two Neural Link e2e specs run nowhere, and both have rotted'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T15:21:06Z'
updatedAt: '2026-09-18T16:15:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/160'
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
closedAt: '2026-09-18T16:15:16Z'
---
# Two Neural Link e2e specs run nowhere, and both have rotted

## Context

Found while running the pin-11 tiers (#157): with the Brain runtime root set, two e2e specs were red that no recorded battery lists. They are red because nothing ever runs them. Re-measured on dev@a94bbd1183, `NEO_AGENTOS_RUNTIME_ROOT` set, `--workers=1`: 2 failed.

- `agentos/AccountsConfigSurface.spec.mjs:81` — `expect(await page.locator('.fm-chip').count()).toBe(listHarnessTypes().length)`: `Expected: 8`, `Received: 9`.
- `agentos/FleetNavFamilyPin.spec.mjs:41` — `nav-family-dark-keeper`: 600 pixels differ from a golden of 2026-08-27; the keeper rail has carried a fifth tab since the System view landed on 2026-09-05.

## The Problem

Two rules decide which e2e specs need the Brain fixture, and they disagree. `playwright.config.e2e.mjs` decides by CONTENT: a spec that names `neuralLink` (`discoverExternalBrainSpecs`) is ignored unless `NEO_AGENTOS_RUNTIME_ROOT` is absolute. `npm run test-e2e:nl` decides by FILE NAME: `agentos/(.*NL|FleetGridKeyboardA11y)`.

Mechanical diff on dev@a94bbd1183: 31 specs name `neuralLink`; two of them miss the name pattern — the two above. So the isolated CI job ignores them (no runtime root), `test-e2e:nl` does not select them, and the Brain-contract CI job only lists (`npm run test-e2e -- --list`). `FleetGridKeyboardA11y` is already spelled out in the pattern: the name rule has drifted before, and the next spec that is not called `*NL` orphans the same way.

`AccountsConfigSurface` is the only e2e over the accounts configuration surface (cold load, save, add a resident, rehydrate over the real Fleet wire); `FleetNavFamilyPin` pins the chrome-tier nav family in both skins. Both went red without anyone seeing it.

## The Architectural Reality

- `test/playwright/playwright.config.e2e.mjs` — `discoverExternalBrainSpecs(root)` (the content rule) feeds `testIgnore` when no runtime root is bound.
- `package.json` — `test-e2e:nl`: `playwright test -c test/playwright/playwright.config.e2e.mjs "agentos/(.*NL|FleetGridKeyboardA11y)" --workers=1` (the config already sets `workers: 1`).
- `.github/workflows/ci.yml` — "Collect the full E2E contract" runs `npm run test-e2e -- --list` with the runtime root: collection, never execution.
- The inverse drift is harmless: `FleetCockpitBarCompositionNL` and `FleetManagerNoTourWitnessNL` match the name pattern without naming `neuralLink`; under the content rule they belong to the isolated run, which already collects them.

## The Fix

- `test-e2e:nl` selects by the config's own rule: a sibling config whose `testMatch` is `discoverExternalBrainSpecs(e2eRoot)` over the base e2e config — one rule, no name convention, no env syntax in an npm script.
- `AccountsConfigSurface`: read which ninth `.fm-chip` the page now renders, and scope the count to the harness group if the chip is legitimate (a product defect if it is not).
- `FleetNavFamilyPin`: re-render its goldens from the spec's own run, by-eye read.
- A unit arm over the two configs: every spec that names `neuralLink` is matched by the Neural Link config and ignored by the unbound base config.

## Acceptance Criteria

- [ ] `npm run test-e2e:nl` runs exactly the specs the e2e config classifies as external-Brain, the two orphans included; no file-name pattern remains in the script.
- [ ] A unit arm fails when a spec names `neuralLink` and the Neural Link selection misses it — derived from the discovery function, not from a list.
- [ ] `AccountsConfigSurface` and `FleetNavFamilyPin` are green on a host with the Brain runtime root; re-rendered goldens come from the spec's own full run and are read by eye.
- [ ] The ninth chip is named in the PR: legitimate (selector scoped) or a defect (filed).

## Out of Scope

Executing the Neural Link battery in CI — it needs a Brain runtime, a display for the platform goldens, and its own time budget (the engine's 2026-09-06 e2e wiring decision: own pipeline, five minutes). Renaming specs to fit a pattern.

## Related

#157 (where the two reds surfaced); #11 (the visual baseline harness — the stamp may list `FleetNavFamilyPin`'s goldens as inputs).

Live latest-open sweep: all 17 open issues of neomjs/neo-agent-institution at 2026-09-18T15:20Z — none equivalent. A2A in-flight claim sweep: the last hour carries no claim on the Institution's e2e selection; the sighting itself is my defect-note of 14:30Z. Memory Core rationale sweep (`query_raw_memories`, the symptom's nouns): the engine-side precedent of 2026-09-06 — an absent runner hid three live e2e failures on dev — and the operator's constraint for CI e2e (own pipeline, ≤5 minutes, no GPU); nothing decided the Institution's Neural Link selection. Own-assignment sweep: #127, #129, #10 — other surfaces.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "orphan e2e specs neuralLink test-e2e:nl name pattern content rule"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59

## Timeline

- 2026-09-18T15:21:06Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T15:21:08Z @neo-fable-clio added the `bug` label
- 2026-09-18T15:21:08Z @neo-fable-clio added the `ai` label
- 2026-09-18T15:21:08Z @neo-fable-clio added the `testing` label
- 2026-09-18T15:30:31Z @neo-fable-clio cross-referenced by #161
- 2026-09-18T15:34:00Z @neo-fable-clio cross-referenced by PR #162
- 2026-09-18T15:46:37Z @neo-fable-clio cross-referenced by #163
- 2026-09-18T16:03:26Z @neo-fable-clio cross-referenced by PR #165
- 2026-09-18T16:15:16Z @tobiu referenced in commit `66bfcf9` - "Merge pull request #165 from neomjs/agent/160-nl-battery-content-rule

test(e2e): the Neural Link battery selects by the config's content rule (#160)"
- 2026-09-18T16:15:16Z @tobiu closed this issue

