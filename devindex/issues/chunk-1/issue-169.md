---
id: 169
title: Upgrade to neo.mjs 14.0.0 with URL worker switch and mobile benchmark HUD
state: OPEN
labels:
  - enhancement
  - ai
  - grid
  - javascript
assignees:
  - neo-gemini-julia
createdAt: '2026-11-26T01:58:14Z'
updatedAt: '2026-11-26T02:54:33Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/169'
author: neo-gemini-julia
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
# Upgrade to neo.mjs 14.0.0 with URL worker switch and mobile benchmark HUD

## Context

`neomjs/neo#22218` (`#issuecomment-7179980304`, `#issuecomment-7180063331`) and `neomjs/neo#22209` (`PR neomjs/neo#22215`, ADR 0043) establish `neomjs/devindex` as the real-application benchmark arm for deciding whether `useVdomWorker` stays `true` (with `#22218` dual-barrier direct `vnode` delivery) or flips to `false`:
1. `DevIndex` exercises every grid scaling axis simultaneously: `50,000` developer records streaming progressively via `StreamConnection` + `StreamParser` (`apps/devindex/store/Contributors.mjs`), `36` rich columns (`OffscreenCanvas` `Activity` sparklines, `Heuristics`, `githubUser`, `githubOrgs`, `17` heatmap year columns), live `Buffer Row Range` / `Buffer Column Range` controls, and heavy post-stream `App`-worker column re-sorts + `50k` structural rank recalculation (`Hireable Only`, `Hide Commit Ratio > 90%`).
2. In addition to Desktop `1x` + `4x` throttled Playwright runs, the pre-registered decision rule (`neomjs/neo#22218#issuecomment-7180063331`) adds a **physical iPhone Safari arm** (`2` performance + `4` efficiency cores), run as two Safari tabs on one phone (`?bench=1&useVdomWorker=true` vs `?bench=1&useVdomWorker=false`) with an on-device timing readout.

Currently in `neomjs/devindex`:
- `package.json` pins `"neo.mjs": "^13.4.0"`, and `MainContainerController.mjs:184` assigns `grid.body.selectionModel`, which throws on `neo.mjs@14.0.0` (`src/grid/Body.mjs:725`).
- `src/MicroLoader.mjs` loads `./neo-config.json` with no URL-parameter override for `useVdomWorker`, preventing side-by-side two-tab arm testing on a physical phone without Playwright network interception.
- `ControlsContainer.mjs:208,222` caps `Buffer Row Range` and `Buffer Column Range` at `'5'`, and there is no on-device `?bench=1` benchmark HUD for Safari mobile runs.

- `Live latest-open + closed sweep:` checked latest 20 open issues (`3` open: `#135`, `#61`, `#9`, green band) + open and closed search (`useVdomWorker OR benchmark OR v14`, `0` hits) at `2026-11-26T01:56:39Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 messages across all read states (`2026-11-26T00:09Z`–`01:55Z`); `@neo-opus-vega` explicitly requested `@neo-gemini-julia` fold this `devindex` prep in (`MESSAGE:de689651`, `MESSAGE:100ff0f9`), no competing claim.
- `MC sweep:` `query_raw_memories` (`"devindex useVdomWorker iPhone Safari scroll benchmark MicroLoader"`, 5 results) + `ask_knowledge_base` (`type: "ticket"`, 5 results); no prior duplicate or contrary decision found.
- `Own-assignment sweep:` `0` open issues assigned to `@neo-gemini-julia` in `neomjs/devindex`.
- `Ownership:` self-assigned (`@neo-gemini-julia`) — active critical-path benchmark delivery for `neomjs/neo#22209` / `#22218`; curated `good first issue` / `help wanted` labels intentionally omitted.

---

## The Problem

1. **`neo.mjs@14.0.0` Compatibility Gap:**
   - `DevIndex.view.home.MainContainerController#onSelectionModelChange` (`MainContainerController.mjs:184`) writes `this.getReference('grid').body.selectionModel`, which throws on `neo.mjs@14.0.0` because `Row` Radio `checked: true` fires during construction and `Neo.grid.Body` moved `selectionModel` ownership to `Neo.grid.View` (`grid.view.selectionModel`).
2. **No Page-Readable Worker-Arm Switch or On-Device Mobile Benchmark Readout:**
   - Physical iPhone Safari runs cannot use Playwright `page.route('**/neo-config.json')`. Without reading `?useVdomWorker=true|false` in `src/MicroLoader.mjs` before `import(d.mainPath)` and exposing an opt-in `?bench=1` Main-thread HUD (`MutationObserver` TVS + `requestAnimationFrame` frame `p95`/`maxRAF` + `Neo.worker.Manager` flight/relay telemetry), two-tab physical-phone benchmarking cannot verify topology or report per-step `p50`/`p95`/`p99` timings.

---

## The Architectural Reality

- **Boot Contract (`src/MicroLoader.mjs`, `dist/production/apps/devindex/index.html`):** Both `development` and `dist/production` load `../../src/MicroLoader.mjs` before importing `d.mainPath`. Parsing `URLSearchParams(globalThis.location?.search)` inside `MicroLoader.mjs` overrides `d.useVdomWorker` deterministically across dev, prod, Desktop Playwright, and Mobile Safari.
- **v14 Multi-Body Grid View (`neo/src/grid/Body.mjs:725`, `neo/src/grid/View.mjs`):** Selection model configuration and events live on `grid.view`, while `bufferRowRange` and `bufferColumnRange` remain reactive configs on `grid.body` (`ControlsContainer.mjs:208,222`).

---

## The Fix

1. **Upgrade `neomjs/devindex` to `neo.mjs@14.0.0`:**
   - Bump `package.json` to `"neo.mjs": "14.0.0"` (with `v14` build devDependencies `highlight.js`, `postcss-selector-parser`, `postcss-value-parser`), migrate `MainContainerController.mjs` (`lines 66, 184`) to `grid.view`, and extend `ControlsContainer.mjs` `Buffer Row Range` (`['1', '2', '3', '5', '15', '25']`) and `Buffer Column Range` (`['1', '2', '3', '5', '10', '15']`).
2. **Add `?useVdomWorker=true|false` URL Override & Opt-In `?bench=1` On-Device Benchmark HUD:**
   - Update `src/MicroLoader.mjs` to parse `?useVdomWorker=true|false` onto `Neo.config.useVdomWorker` before `import(d.mainPath)` and lazily load an opt-in Main-thread benchmark HUD when `?bench=1` is present (`zero` overhead when absent).
   - The `?bench=1` HUD verifies live `Neo.config.useVdomWorker` + `Neo.worker.Manager.getWorker('vdom')`, records initial `50k` stream-in telemetry, and provides one-tap execution + Markdown copy for `n = 20` 2D scroll hops and `50k`-record sort/rank-filter operations (usable both on physical iPhone Safari and automated Playwright runs).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `?useVdomWorker=true\|false` (`src/MicroLoader.mjs`) | `apps/devindex/neo-config.json` & `Neo.worker.Manager` | Overrides `Neo.config.useVdomWorker` prior to `Main.mjs` boot in dev & prod | Omitting param preserves engine/app default | Ticket / PR | Playwright topology check (`true` & `false`) |
| `?bench=1` HUD (`apps/devindex/resources/device-benchmark.mjs`) | `neomjs/neo#22218#issuecomment-7180063331` | Renders mobile-friendly HUD measuring stream, `n=20` scroll TVS/frame `p95`/`p99`, `50k` sort/filter, and worker flights | Not loaded when `?bench=1` is absent | Ticket / PR | Desktop `1x` + `4x` 3-way run + iPhone-ready URL |

- **Decision Record impact:** `aligned-with` `neomjs/neo#22218` / ADR 0043 evaluation gate.

---

## Acceptance Criteria

- [ ] **AC-1:** `neomjs/devindex` builds cleanly (`npm run build-all -- -l no`) on `neo.mjs@14.0.0`, with `MainContainerController` migrated to `grid.view.selectionModel` and `Buffer Row/Column Range` extended up to `25` / `15`.
- [ ] **AC-2:** Navigating with `?useVdomWorker=true` vs `?useVdomWorker=false` deterministically boots `3`-thread (`workers.vdom` present) vs `2`-thread (`workers.vdom` absent) topologies on `dist/production` without Playwright route interception.
- [ ] **AC-3:** Navigating with `?bench=1` mounts a mobile-usable benchmark HUD reporting verified worker topology, `50k` stream telemetry, `n = 20` scroll TVS (`mean`, `p50`, `p95`, `p99`, `max`, `frame p95`, `max RAF`), `50k` sort/rank-filter timings, and one-tap Markdown copy, while omitting `?bench=1` loads zero benchmark UI.

---

## Out of Scope

- Modifying `DevIndex` spider/opt-in backend services (`services/`).
- Changing `DefaultConfig.useVdomWorker` in `neo.mjs` (`neomjs/neo#22209`).

## Avoided Traps

- **Relying on Playwright `page.route` for mobile Safari:** Physical iPhone Safari tabs cannot use Playwright route interception; `MicroLoader.mjs` URL-param parsing works natively in any browser tab.
- **Loading benchmark HUD unconditionally:** Gating dynamic `import()` on `?bench=1` keeps normal `DevIndex` sessions completely untouched.

## Related

- `neomjs/neo#22218` · `neomjs/neo#22209` (`PR neomjs/neo#22215`) · `neomjs/benchmarks#15`

Origin Session ID: f5dc9080-2ae6-4908-87cf-b91fa9ff9ab5
Retrieval Hint: `"useVdomWorker" "bench=1" "MicroLoader" "devindex" "iPhone Safari" "22218"`

## Timeline

- 2026-11-26T01:58:15Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-26T01:58:17Z @neo-gemini-julia added the `enhancement` label
- 2026-11-26T01:58:17Z @neo-gemini-julia added the `ai` label
- 2026-11-26T01:58:17Z @neo-gemini-julia added the `grid` label
- 2026-11-26T01:58:18Z @neo-gemini-julia added the `javascript` label
- 2026-11-26T02:54:32Z @neo-gemini-julia cross-referenced by PR #171
- 2026-11-26T03:54:25Z @neo-gemini-julia referenced in commit `c354a42` - "fix(devindex): exclude per-column warm-up sort from benchmark timing (#169)"

