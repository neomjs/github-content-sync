---
id: 193
title: The harness theme builds drop the cockpit's rows from the theme map
state: CLOSED
labels:
  - bug
  - ai
  - build
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T10:37:45Z'
updatedAt: '2026-09-25T11:32:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/193'
author: neo-fable-clio
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T11:32:37Z'
---
# The harness theme builds drop the cockpit's rows from the theme map

## Context

Operator observation, 2026-09-25 (a `start:brain` window from this checkout): *custom theming is completely missing inside the electron shell*. The cockpit rendered with the engine's default component looks — none of the Fleet Manager's own SCSS (`resources/scss/src/apps/agentos/**`, `resources/scss/theme-neo-*/apps/agentos/**`) reached the window.

## The Problem

The App worker inserts a class's CSS only when `resources/theme-map.json` has a row for it (`neo.mjs/src/worker/App.mjs#insertThemeFiles`: app classes resolve as `apps.<app>.<path>` against `cssMap.fileInfo`; no row → no `<link>`). The shell serves that map from the product root (`contentPolicy.mjs` allowlists `/resources/theme-map.json`), so whatever the harness's own asset preparation writes there is what the cockpit gets. Two writers in `harness/` produce a map without `apps.agentos` rows:

1. **`pack.mjs` runs the engine theme builder with `-f`** (`pack.mjs:682`: `themes.mjs -f -n -e dev -t all`). `-f` is `--framework` (`themes.mjs:39`), the "build inside the neo repository" switch: with it, `getAllScssFiles` reads only the engine's `resources/scss`, never the workspace's (`themes.mjs`, the `insideNeo` branch), so the workspace's CSS is not rebuilt and the regenerated map carries engine rows only. Measured after a pack run on this checkout: 249 rows, **0** `agentos` rows. The comment above the call reads the flag as "force/rebuild" — it is not.
2. **`prepareAssets.mjs` rebuilds one theme per run** (`buildTheme('theme-neo-dark')`, then `'theme-neo-light'`). The engine builder regenerates the map from the engine seed plus the themes it was asked for (`themes.mjs:315-322`, "full builds REGENERATE the map"), so a single-theme run keeps workspace rows only for that theme: measured, `-t theme-neo-dark` alone → 34 `agentos` rows, all `src|theme-neo-dark`, 0 carrying `theme-neo-light`. After the harness's dark→light sequence the cockpit's dark rows are gone, and the cockpit runs dark.

Falsifier: `npm run build-themes -- -n -e dev -t all` on the same checkout → 283 rows, 34 `apps.agentos.*` rows, e.g. `apps.agentos.Viewport → src|theme-neo-dark|theme-neo-light`.

## The Architectural Reality

- `harness/prepareAssets.mjs` is the `pre*` step of every harness script (`start`, `smoke*`, `witness:lifecycle`, `dist`); it owns existence + staleness of the dev assets and calls `npm run build-themes` per theme. It has no spec and runs its work at module top level.
- `harness/pack.mjs#stageOrganism` rebuilds themes before copying `dist/development/css` + `resources/theme-map.json` into the stage (`pack.mjs:679-682`), so the packaged cockpit inherits the same engine-only map.
- The engine builder's `-t <one theme>` is a speed switch for the engine repository; in a workspace it is lossy for the map by construction. The consumer-side rule is therefore one `-t all` build; an engine-side merge for single-theme workspace builds is a separate question (noted, not filed).

## The Fix

- `pack.mjs`: drop `-f`; the theme build args become one exported constant shared with `prepareAssets.mjs` (`-n -e dev -t all`).
- `prepareAssets.mjs`: one `-t all` build replaces the per-theme loop (both the stale-rebuild and the missing-asset paths); the module guards its top-level run behind the entry-module check so it is importable; `buildThemes` takes an injected `spawnFn`.
- Specs: a new `prepareAssets.spec.mjs` pins the single spawn and its argv (no `-f`, `-t all`); `pack.spec.mjs` pins that pack uses the same constant.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `prepareAssets.mjs#buildThemes` | the engine builder's CLI (`themes.mjs` options) | one `npm run build-themes -- -n -e dev -t all` | unchanged: a non-zero exit rejects with the code | `harness/README.md` Run (unchanged commands) | spec + the map row counts above |
| `pack.mjs` theme build (`:682`) | same | same argv, no `-f` | unchanged | inline comment corrected | spec |

## Decision Record impact

`aligned-with ADR 0034` (the vessel wraps the built product; the product's own theme is part of what it wraps).

## Acceptance Criteria

- [ ] Red-first: `prepareAssets.spec.mjs` fails on `dev` (two spawns, per-theme argv) and passes with one spawn carrying `['run', 'build-themes', '--', '-n', '-e', 'dev', '-t', 'all']`.
- [ ] `pack.spec.mjs` pins pack's theme-build argv to the shared constant (no `-f`).
- [ ] After `node harness/prepareAssets.mjs` on a checkout with stale css, `resources/theme-map.json` carries `apps.agentos.*` rows with both themes (receipt: row counts before/after).
- [ ] A `start:brain` / `npm start` window on this machine shows the cockpit's own theming (operator's eyes as the witness).
- [ ] The `test/playwright/unit/harness/` tree stays green.

## Out of Scope

- The engine builder's single-theme semantics in workspaces (a consumer trap; noted for the engine, not filed here).
- The design pass itself (which default-theme overrides remain) — #13 / #24.

## Related

#7 (parent) · #13 (design conformance) · #12 · #191 (the other boot defect found today)

Live latest-open sweep: all 16 open issues checked at 2026-09-25 10:36Z; no equivalent. A2A claim sweep: no claim on the harness theme build. Memory Core sweep: no prior decision on the harness's theme argv.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "harness theme-map apps.agentos rows missing -f --framework prepareAssets per-theme"

## Timeline

- 2026-09-25T10:37:46Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T10:37:47Z @neo-fable-clio added the `bug` label
- 2026-09-25T10:37:47Z @neo-fable-clio added the `ai` label
- 2026-09-25T10:37:48Z @neo-fable-clio added the `build` label
- 2026-09-25T10:37:48Z @neo-fable-clio added the `design` label
- 2026-09-25T10:38:12Z @neo-fable-clio added parent issue #7
- 2026-09-25T10:43:00Z @neo-fable-clio cross-referenced by PR #194
- 2026-09-25T10:54:24Z @neo-fable-clio cross-referenced by #195
- 2026-09-25T11:00:54Z @neo-opus-grace cross-referenced by #13
- 2026-09-25T11:23:54Z @neo-fable-clio referenced in commit `e5852e7` - "fix(harness): the stage runner is a seam, and the theme-build call site is pinned (#193)

Grace's review: the spec pinned themeBuildArgv()'s output, but no test reached
stageOrganism, so the old -f literal at the call site would have stayed green.
stageOrganism now takes runFn (default: the execFileSync runner) for every child
process it issues, and pack.spec drives it against the scaffolded roots with a
recording runner that stops after the first call: that call must be node with
themeBuildArgv(enginePackageRoot) in the product root. Control: restoring the -f
literal at the call site fails the arm."
- 2026-09-25T11:32:37Z @tobiu closed this issue
- 2026-09-25T11:32:37Z @tobiu referenced in commit `d2d9f86` - "Merge pull request #194 from neomjs/fix/193-theme-map-all

fix(harness): the theme builds run once for every theme, without the framework switch (#193)"

