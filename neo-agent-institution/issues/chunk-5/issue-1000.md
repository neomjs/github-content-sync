---
id: 1000
title: Create door and setup fixtures stall on recipe v2 desktop-wake steps
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-16T09:18:10Z'
updatedAt: '2026-11-16T09:18:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1000'
author: neo-gemini-maryam
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
# Create door and setup fixtures stall on recipe v2 desktop-wake steps

## Context

After `neomjs/neo-agent-brain#1780` (`neomjs/neo-agent-brain#1738`) merged to `neo-agent-brain@dev`, `Institution CI` fails on every PR targeting `dev` (first observed on `#999`, run `43833453559`, job `133184141279` and `Isolated Institution`): `npm run resolve-org-dev` upgrades `neo-agent-brain` from pinned `fef1702` (`RECIPE_VERSION = 1`) to `dev` (`RECIPE_VERSION = 2`), failing **22** pre-existing unit tests (`11` in `test/playwright/unit/apps/agentos/view/setup/createContainer.spec.mjs` and `11` in `test/playwright/unit/harness/setupBroker.spec.mjs`). `#658` established the matching precedent when `Brain #960` added step 13.

## The Problem

`Brain #1780` moved the shared first-run recipe (`firstRunRecipe.mjs`) to `RECIPE_VERSION = 2` (15 steps):

1. Added question `desktop-wake` (`'enabled' | 'declined'`, pending on `darwin`, auto-`ok` `skipped: true` on non-Darwin) and effect `start-wake-receiver` (`gates: ['served-plane']`).
2. Updated `observeEnvCarrier` (`ai/scripts/setup/firstRun.mjs:211-240`) to require `NEO_WAKE_RECEIVER_RECORDS_HOST_DIR=<stateDir>/records` and an owner-only `0700` `records` directory.
3. Updated `observeRunningCompose` (`ai/scripts/setup/firstRun.mjs:277-399`) to inspect the running `mc-server` `/app/wake-receiver-records` bind mount via `docker ps` + `docker inspect`.

Meanwhile, in `neomjs/neo-agent-institution`:

- `apps/agentos/model/SetupStep.mjs` omits `skipped` and `capability`, dropping those wire fields during store projection.
- `apps/agentos/util/SetupAsks.mjs` and `apps/agentos/view/setup/CreateContainer.mjs` only model `preset` and `provider-key` in `Where it runs`, so on `darwin` after answering both, `desktop-wake` stays `pending`, every effect sets `waitsFor: 'desktop-wake'`, and `SetupAsks.describeAsks` leaves `startRow: null` (UI stall). On non-Darwin (`skipped: true`), `StepList.mjs` offers a bogus `change` chip on `desktop-wake`.
- `harness/setupBroker.mjs:215, 298, 482` and `test/playwright/fixture/pinnedSetupHost.mjs:125` call `cli.hostLayout({stateRoot})` without `homeDir`, causing Darwin tests to probe/write `~/Library/LaunchAgents/com.neomjs.agent-os-wake.plist` under the real home directory.
- `pinnedSetupHost.mjs` leaks `process.platform`, returns empty stdout for `docker ps`/`docker inspect` while logging observer probes into `world.commands`, and omits scripted `launchctl` and wake listener probes; `setupBroker.spec.mjs` and `createContainer.spec.mjs` hardcode v1 progress/consents/effects (`package.json` is also still pinned at `fef1702`).

## The Architectural Reality

Design authority: ADR 0041 §2.10 ("The recipe's step list is the one order… An effect row that is not `ok` names as data (`waitsFor`) the first earlier question, effect or declared gate that is not `ok`") and `AgentOS.view.setup.CreateContainer` JSDoc ("every row is the vessel's fresh evaluation… the three blocks read their rows").

- `package.json:70` & `.github/workflows/ci.yml:40, 62`: CI runs `npm run test-unit` first against `package-lock.json` and then after `npm run resolve-org-dev`, so the committed Brain pin and `resolve-org-dev` must both align on `RECIPE_VERSION = 2`.
- `apps/agentos/view/setup/CreateContainer.mjs` is currently 981 lines (`checkAppFileSizes.mjs` errors past 1,000 lines); pure text/block helpers belong in `AgentOS.util.SetupAsks`.
- `apps/agentos/view/setup/StepList.mjs:55-66`: `actionsFor` derives row chips from `{id, kind, status, waitsFor, exits, answer, capability}`.
- `harness/setupBroker.mjs:144` & `test/playwright/fixture/pinnedSetupHost.mjs:65`: host seam for `cli.hostLayout({stateRoot, homeDir})`.

## The Fix

1. Advance `"neo-agent-brain"` in `package.json` + `package-lock.json` to current `neomjs/neo-agent-brain@dev`.
2. Update `SetupStep.mjs` (`skipped`, `capability`), `SetupAsks.mjs` (`describeAsks` accounting for `desktop-wake`, plus pure `budgetText`, `ledeText`, `describeWhere` helpers extracted from `CreateContainer.mjs` to keep it below 975 LOC), `StepList.mjs` (`enable` / `decline` chips for `desktop-wake`, empty when `capability?.supported === false`), and `CreateContainer.mjs` (keep 3 `.fm-setup-ask` blocks; render desktop-wake consent controls inside `Where it runs` when needed and toggleable on `Change`; dispatch `setupAnswer` on `desktop-wake`).
3. Add optional `homeDir` to `createSetupBroker` (`harness/setupBroker.mjs`) and `pinnedSetupHost.mjs` (`homeDir = tempDir()`, `platform = 'darwin'`), script `docker ps`/`inspect`, `launchctl`, and `probeWakeListener` cleanly without pushing observer reads into `world.commands` or touching real `~/Library/LaunchAgents`.
4. Update `setupBroker.spec.mjs`, `createContainer.spec.mjs`, `FleetSetupCard.spec.mjs`, and `FleetCockpitVisual.spec.mjs` (adding Darwin `declined` and Linux `platform: 'linux'` unit arms), re-capture Darwin setup goldens via `test-goldens`, and re-stamp `baseline-inputs.txt`.

## Acceptance Criteria

- [ ] `package.json` / `package-lock.json` pin current `neomjs/neo-agent-brain@dev`, and both locked (`npm run test-unit`) and `resolve-org-dev` unit runs pass with `0` failures.
- [ ] `CreateContainer` + `SetupAsks` + `StepList` unblock both Darwin (`enabled` / `declined`, toggleable on `Change`) and non-Darwin (`skipped`, no bogus action chip) flows while keeping `.fm-setup-ask` count at `3` and `CreateContainer.mjs` under `check-app-file-sizes` (`<= 1000` LOC).
- [ ] `harness/setupBroker.mjs` and `test/playwright/fixture/pinnedSetupHost.mjs` thread `homeDir` into `cli.hostLayout` so no test touches real `~/Library/LaunchAgents`, and observer reads (`docker ps`, `docker inspect`, `launchctl print`) never pollute `world.commands`.
- [ ] Isolated `test-unit`, `test-components`, `CI=1 test-e2e` (`FleetSetupCard.spec.mjs` all green), Darwin visual setup capture (`0` pixel diffs), `check-visual-baselines`, and both CI jobs (`Isolated Institution`, `Explicit Brain contract`) pass.

## Out of Scope

- `#997` deliverables: `harness/pack.mjs` `deploy` tree staging, `runtimeGeneration` retention, `devFleetServer` post-setup live wake receiver refresh, and Remote Connect return-route verification.

## Related

- `neomjs/neo-agent-brain#1738` / `neomjs/neo-agent-brain#1780` (upstream recipe v2)
- `#997` (remaining packaged/live Fleet wake pickup work)
- `#999` (first PR hit by the CI regression)
- `#658` (precedent recipe-count drift repair)

Decision Record impact: aligned with ADR 0041 §2.10.

Live latest-open + closed sweep: checked latest 20 open of 60 total open issues (red band; scoped CI-unblock bug needed beside `#997`) + open & closed search (`CreateContainer OR setupBroker OR desktop-wake`, read closed `#658` and `#547` Out of Scope / Avoided Traps) at 2026-11-16T09:14Z; no equivalent open ticket found. A2A in-flight claim sweep: latest 30 messages across all read-states (no competing claim or overlap in herd window). MC sweep: `"CreateContainer setupBroker 22 failed Explicit Brain contract desktop-wake"`, 5 results, no prior decision found besides `#658` pattern. Own-assignment sweep: 2 open (`#989`, `#920`), none overlapping. Structure map: ran `npm run ai:structure-map -- --files --loc` in Brain (`exit 0`); no new `.mjs` file placed.

Origin Session ID: 76097b8c-6818-4c1f-a0c1-abf162c6100c
Retrieval Hint: "Institution CreateContainer SetupAsks pinnedSetupHost recipe v2 desktop-wake start-wake-receiver CI failure"

— Maryam 🧭

## Timeline

- 2026-11-16T09:18:11Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-16T09:18:12Z @neo-gemini-maryam added the `bug` label
- 2026-11-16T09:18:13Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-16T09:18:13Z @neo-gemini-maryam added the `ai` label
- 2026-11-16T09:18:13Z @neo-gemini-maryam added the `regression` label

