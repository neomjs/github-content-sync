---
id: 153
title: build-all runs in the engine's framework mode and compiles every engine app
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T13:03:34Z'
updatedAt: '2026-09-18T13:38:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/153'
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
closedAt: '2026-09-18T13:38:51Z'
---
# build-all runs in the engine's framework mode and compiles every engine app

## Context

@neo-opus-ada measured this while building neomjs/neo#18868 (PR neomjs/neo#18872) and asked over A2A (2026-09-18) whether it is intended. It is not a recorded decision. The manifest arrived whole in fac6a0c (2026-08-26) as the engine's own script block re-pointed at `./node_modules/neo.mjs/…`, and the engine's `build-all` / `build-themes` / `build-threads` carry `-f`. 86afc89 (`#3`, the consumer scss build) took `-f` off `build-themes`; the other two kept it.

## The Problem

`-f, --framework` sets `insideNeo = true` in the engine's `all.mjs` and `buildThreads.mjs`. Run from this workspace, that makes the App worker's `importApp` context reach the engine's own apps and examples — 153 of them by Ada's packed-and-installed probe, the portal views among them.

- **Footprint, measured at pin 10 (engine dev@70c2c94618), `npm run build-all`:** `dist/production` 7,739 files / 60 MB / 1,319 js / 314 App-worker chunks; `dist/development` 324 App-worker chunks. The consumer-mode control below compiled **35**.
- **Robustness:** an engine app that cannot compile from an installed copy reddens this repo's build. neomjs/neo#18849 hit pin 10 for that reason alone; a scaffolded workspace never compiles the portal views.
- The worker contexts skip the consumer rebasing neomjs/neo#17882 added.

- **The product app is not in its own production build** (measured 2026-09-18 while fixing this). In framework mode the `importApp` context stays rooted in the engine package, so `dist/production/appworker.js` maps `apps/portal/app.mjs` and **not** `apps/agentos/app.mjs`; `dist/production/apps/agentos/index.html` stays at 11 DOM nodes (fresh origin, fronted tab, no failed request — the miss is inside the App worker). The consumer-mode bundle maps `apps/agentos/app.mjs`, not the portal, and boots: 571 nodes, no console message.

What it does **not** cost: the packaged artifact. `harness/pack.mjs` copies `dist/development/css` only.

## The Architectural Reality

- `package.json:41`, `:42`, `:46` — `build-all`, `build-all-questions`, `build-threads` pass `-f`.
- **Control, 2026-09-18, clean `dist`:** `node ./node_modules/neo.mjs/buildScripts/build/all.mjs -n` → **rc=1 after 9 s**. Four thread builds compile; the `service` entry fails with `Module not found: Can't resolve '<workspace>/ServiceWorker.mjs'`. Consumer mode resolves the service-worker entry from the workspace root, where the scaffold writes one; this repo was not scaffolded and has none. No `neo-config.json` here sets `useServiceWorker`.
- So dropping the flag alone turns the build red: the fix is the flag **and** the entry consumer mode expects.

## The Fix

1. Add the workspace-root `ServiceWorker.mjs` the scaffold writes (read the engine's `buildScripts/create/app.mjs` for its exact shape), or — if the engine should not demand one from a workspace whose apps never enable it — take that to the engine instead and say so in the PR.
2. Drop `-f` from `build-all`, `build-all-questions`, `build-threads`.
3. Prove the result: `build-all` rc=0 twice from a clean `dist`; `apps/agentos` boots from `dist/production` (the docs app: same state before and after, see the ACs); `npm run dist` / the pack stage unchanged; both footprints in the PR body.

## Acceptance Criteria

- [ ] No Institution build script passes `-f`.
- [ ] `npm run build-all` is green twice in a row from a clean `dist`.
- [ ] `apps/agentos` boots from `dist/production` with no page or worker error. The docs app is out of reach here: it fails identically in both modes (`Cannot find module './docs/app.mjs'` — the engine's `webpackInclude` for `importApp` cannot match a `docs/app.mjs` entry, defect-noted to the engine), so this ticket neither breaks nor fixes it.
- [ ] The App-worker chunk count and `dist/production` size are recorded before and after.
- [ ] The pack stage (`harness/pack.mjs`) still produces the same artifact.
- [ ] If consumer mode turns out unable to build this workspace for a reason beyond the service-worker entry, the PR stops and the finding goes to the engine with the log.

## Out of Scope

- The engine-side observer that compiles every shipped app from an installed copy (neomjs/neo#18868) — it uses framework mode as an instrument, deliberately.
- Whether consumer mode should require a root `ServiceWorker.mjs` at all — an engine question, raised only if step 1 shows it.

## Avoided Traps

- Flipping the two flags and calling it done: the control shows that build is red.
- Reading the smaller chunk count of the aborted control as the final number — it is a lower bound from a build that stopped at the fifth thread.

## Related

neomjs/neo#18849, neomjs/neo#18851, neomjs/neo#18868, neomjs/neo#17882; #151 (the pin that surfaced it).

Decision Record impact: none.

Live latest-open sweep: all 18 open issues read at 2026-09-18T13:02:39Z, plus `gh issue list --state all --search 'build-threads OR "framework mode" OR insideNeo'` → no hit. A2A in-flight sweep: newest inbox rows to 12:57Z, no claim on this surface; the origin is Ada's direct question.
MC sweep: `neo-agent-institution build-all build-threads framework mode -f insideNeo App worker bundles every engine app and example, consumer workspace build`, 5 results, all session-boot noise, no prior decision found.
Own-assignment sweep: 4 open here (#127, #128, #129, #10), none overlapping.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59
Retrieval Hint: "Institution build-all framework mode -f insideNeo ServiceWorker.mjs consumer mode control"


## Timeline

- 2026-09-18T13:03:34Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T13:03:35Z @neo-fable-clio added the `bug` label
- 2026-09-18T13:03:36Z @neo-fable-clio added the `ai` label
- 2026-09-18T13:03:36Z @neo-fable-clio added the `build` label
- 2026-09-18T13:26:57Z @neo-fable-clio cross-referenced by PR #154
- 2026-09-18T13:38:51Z @tobiu referenced in commit `9064ee3` - "Merge pull request #154 from neomjs/agent/153-consumer-mode-build

fix(build): the workspace builds in consumer mode, so its production App worker holds the agentos app (#153)"
- 2026-09-18T13:38:52Z @tobiu closed this issue
- 2026-09-18T13:49:33Z @neo-opus-grace cross-referenced by #18889
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157

