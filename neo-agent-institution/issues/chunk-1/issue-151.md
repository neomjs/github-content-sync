---
id: 151
title: 'Engine pin 10 — dev@70c2c94618: 88 commits, theme weight + Monaco build'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T10:40:55Z'
updatedAt: '2026-09-18T12:11:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/151'
author: neo-fable-clio
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 149 Six dev-scope Dependabot alerts and three CodeQL findings stay open'
blocking: []
closedAt: '2026-09-18T12:11:23Z'
---
# Engine pin 10 — dev@70c2c94618: 88 commits, theme weight + Monaco build

## Context

The Institution pins the engine at `github:neomjs/neo#337c9fd5d1` (pin 9, 2026-09-14). Engine `dev` is at `3f859a4803` — 81 commits later (`git log --oneline 337c9fd5d1..origin/dev | wc -l`). Operator direction 2026-09-18: move the pin before Fleet Manager app work resumes. `neo.mjs` is a `github:` ref Dependabot cannot bump (`.github/dependabot.yml`), so the move is manual.

## The Problem

Every FM lane after this one builds on engine behaviour the pin does not carry. The range holds no deleted or renamed `src/` / `resources/scss` file (`git diff --diff-filter=DR`), so the drift is behavioural. The commits a cockpit consumer can feel:

- neo#18747 — every pure engine value sheet declares at `:where()` weight: consumer overrides that won by source order or equal specificity now win by weight, and any FM rule that *relied* on losing changes too. Goldens are the witness.
- neo#18679 — an adopted instance takes its container's theme unless it carries its own: the vessel path (tear-out adoption) is the FM's consumer of exactly this.
- neo#18711 / neo#18708 — grid selection models declare what they select; a swapped-out model leaves nothing painted. The registers and the mailbox grid run `viewConfig: {selectionModel: null}`.
- neo#18722 — a tab projection landing while the overflow menu opens parks instead of rewriting it (the cockpit's tab bars project).
- neo#18769 — grid cell editing becomes a Grid-native session (no FM grid edits today; the pooled-cell render path is shared).
- neo#18717 — `buildScripts/build/all.mjs` now bundles Monaco from its ESM sources, resolving `dompurify` the way npm does. **Blocked by #149**: without that override this step would bundle `dompurify 3.4.8`.

## The Architectural Reality

- `package.json` `dependencies["neo.mjs"]` + the lock's `node_modules/neo.mjs` entry are the pin; the visual-baseline stamp (`test/playwright/visual/__screenshots__/baseline-inputs.json`) hashes the engine lock entry, so the stamp moves with it while the pixels under it may be stale — a pin bump runs `test-visual` and reads every drift by eye.
- `npm install` does not re-extract a `github:` SHA dependency over an existing `node_modules/neo.mjs`; remove it first and verify the install by a code marker only the target carries.
- CI runs the branch's own lock in two jobs: isolated (unit + components + e2e) and Explicit Brain contract (full unit + e2e collection).

## The Fix

1. Drift check first: FM-used engine identifiers/configs ∩ what the range changed.
2. Move the pin + lock; marker-verified install.
3. Follow each real consumer drift in the consumer (specs included); engine defects found on the way go to `neomjs/neo`, not into consumer workarounds.
4. `test-unit` (both modes), `test-components`, `test-e2e`, the NL battery, `test-visual` alone; goldens re-rendered only from a full visual run, each drift read by eye; restamp as the last staged change.
5. `build-all` once at the new pin (the Monaco + mermaid steps resolve from this workspace).

## Acceptance Criteria

- [ ] `package.json` + lock pin `neo.mjs` at the target SHA; a code marker proves the installed tree is that SHA.
- [ ] Unit tier green in both CI modes; components + e2e green in CI.
- [ ] NL battery at the new pin: every red is either fixed, or proven pre-existing by a pin-9 control run with identical errors.
- [ ] Visual tier: every golden drift is named with its cause (theme weight / adopted theme / none) and re-rendered from a full run; stamp fresh.
- [ ] `build-all` completes at the new pin and the Monaco bundle fingerprints exactly one DOMPurify, ≥ 3.4.13.

## Out of Scope

The Brain pin. #128 (roster measured rows — its engine half has been on the pin since pin 9). New FM features.

## Avoided Traps

- Regolden from an isolated `-g` visual run (webfont starvation renders a different frame).
- Calling a full-tier red a pin regression before the arm ran alone ×3 and against a pin-9 control (`npm install --no-save neo.mjs@github:neomjs/neo#337c9fd5d1`).
- Absorbing an engine defect in a consumer constant.

## Related

#149 (blocks this) · #136 (pin 9) · neo#18747 · neo#18679 · neo#18717 · neo#18711 · neo#18722

Decision Record impact: none.

Live latest-open sweep: 20 open issues + an all-state `engine pin in:title` search read at 2026-09-18T10:40Z — prior pins (#81, #90, #98, #136) are closed, none open. A2A sweep: latest 50 messages, no claim on this surface. Own open assignments (#10, #124, #127, #128, #129, #149): no overlap.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "Institution engine pin 10 3f859a4803 where() theme weight adopted instance theme Monaco build step golden drift"

## Timeline

- 2026-09-18T10:40:55Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T10:40:56Z @neo-fable-clio added the `enhancement` label
- 2026-09-18T10:40:56Z @neo-fable-clio added the `ai` label
- 2026-09-18T10:40:57Z @neo-fable-clio added the `dependencies` label
- 2026-09-18T10:57:34Z @neo-fable-clio cross-referenced by #18844
- 2026-09-18T10:58:44Z @neo-opus-grace cross-referenced by PR #150
- 2026-09-18T11:09:19Z @neo-fable-clio cross-referenced by #18849
### @neo-fable-clio - 2026-09-18T11:13:26Z

**Lane state, 2026-09-18 ~11:15Z — measured at `dev@3f859a4803`, branch `agent/151-engine-pin-10` (stacked on #150's branch, not pushed yet).**

| Tier at pin 10 | Result |
|---|---|
| unit, Brain-contract mode / isolated | 910/910 · 812/812 (+2 new arms since) |
| components | 3/3 |
| e2e (the CI selection) | 8 red → 6/8 after the fix below; the 2 left are the 1-px `FleetCockpitBarCompositionNL` host goldens (red identically at pins 8 and 9, PR #137) |
| visual, run alone | 14/14 — no golden moved; the `:where()` theme weight (neo#18747) and the adopted-instance theme (neo#18679) change no cockpit pixel |
| NL battery | 25 green / 12 red: the 6 recorded host reds (same errors as PR #137), my untracked witness rig, and 5 new — all 5 from the engine's new worker-error fixture (neo#18554), none from a scenario assertion |

**The drift is the fixture, and it was right every time.** What it surfaced:

1. **Ours, fixed on the branch** (`be0220d`): `Container#onConstructed` calls `loadOperatorIdentity()` fire-and-forget and its bridge throw ("fleet bearer not injected") escaped as an unhandled App Worker rejection on *every* bearer-less boot — that alone was 8 of 8 e2e reds. And `switchToProfile` wrote `instanceState: 'starting'`, then let `installFleetBridge` refuse a remote endpoint by throwing: unhandled error, switcher stranded in "starting". Both end as verdicts now; two red-first unit arms.
2. **Engine, mine** — neo#18844 / PR neo#18845: `list.plugin.Animate` fades the WRONG rows when one filter pass adds and removes, and throws when the list is rebuilt inside its 50 ms frame (`FleetGridScaleNL`). The roster is a consumer.
3. **Engine, mine** — neo#18849 / PR neo#18851: three portal components import `marked` through a path that leaves the package → `build-all` exits 1 in any consumer from the second build on. Proven pre-merge here: with the three fixed files, `build-all` rc=0 twice; Monaco bundle fingerprints exactly one DOMPurify, 3.4.14.
4. **Engine, @neo-opus-ada's surface** (defect-note + fork sent): seven addon-teardown sites reject `NEO_DEAD_PORT` unhandled when the window they address just died — `FleetCockpitTearOutNL` ×2 and `FleetCockpitNWindowNL`. These three arms stay red-honest locally (outside CI); nothing is masked with `workerErrors.expect`.
5. **Ours, fixed** (`f3e40b3`): a `{…}[]` JSDoc type in `CockpitPerspectives.mjs` stopped `build-all`'s docs step.

**Consequence for this ticket:** AC-5 (`build-all` completes) cannot be met at `3f859a4803` — it needs the SHA that carries neo#18851. The pin target moves there (and takes neo#18845 with it if it has merged); the PR opens once #150 and that SHA exist. `cockpit/Controller.mjs` stands at 997 of 1000 lines — the next change to it starts with a seam cut.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


- 2026-09-18T11:55:23Z @neo-fable-clio cross-referenced by PR #152
- 2026-09-18T11:55:24Z @neo-fable-clio changed title from **Engine pin 10 — dev@3f859a4803: 81 commits, theme weight + Monaco build** to **Engine pin 10 — dev@70c2c94618: 88 commits, theme weight + Monaco build**
- 2026-09-18T12:11:23Z @tobiu referenced in commit `5c94baa` - "Merge pull request #152 from neomjs/agent/151-engine-pin-10

chore(deps): engine pin 10 — dev@70c2c94618, and what the engine's worker-error fixture found on the way (#151)"
- 2026-09-18T12:11:23Z @tobiu closed this issue
- 2026-09-18T13:03:35Z @neo-fable-clio cross-referenced by #153
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

