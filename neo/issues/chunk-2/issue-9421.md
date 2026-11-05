---
id: 9421
title: 'Refactor: Move grid column components into `src/grid/column/component/`'
state: OPEN
labels:
  - no auto close
  - refactoring
  - architecture
  - grid
assignees:
  - tobiu
createdAt: '2026-03-09T16:23:08Z'
updatedAt: '2026-11-05T10:41:32Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9421'
author: tobiu
commentsCount: 2
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
# Refactor: Move grid column components into `src/grid/column/component/`

### Goal
To improve namespace organization and prevent polluting the top-level `src/component/` directory, all components that are exclusively built to be used within grid cells should be moved to a dedicated `src/grid/column/component/` directory.

### Scope
Migrate the following components and their corresponding SCSS files (both in `src` and across all themes):
- `GitHubOrgs`
- `GitHubUser`
- `IconLink`
- `Progress`
- `Sparkline`
- `CountryFlag`

### Tasks
- [ ] Move JS files to `src/grid/column/component/` and update their `className`s to `Neo.grid.column.component.*`.
- [ ] Move SCSS structural files within `resources/scss/src/component/` to `resources/scss/src/grid/column/component/`.
- [ ] Move SCSS theme variables files within `resources/scss/theme-*/component/` to `resources/scss/theme-*/grid/column/component/`.
- [ ] Update all import paths inside the respective column definitions (`src/grid/column/*.mjs`).
- [ ] Update any example apps or tests that might be importing these components directly.


## Timeline

### @github-actions - 2026-06-08T06:11:46Z

This issue is stale because it has been open for 90 days with no activity.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:41:32Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - `src/grid/column/component/Tree.mjs` (`Neo.grid.column.component.Tree`) already lives under `src/grid/column/component/`.
  - The 6 components listed in `#9421` (`CountryFlag.mjs`, `GitHubOrgs.mjs`, `GitHubUser.mjs`, `IconLink.mjs`, `Progress.mjs`, `Sparkline.mjs`) still reside in `src/component/` (imported by `src/grid/column/CountryFlag.mjs:1`, `GitHubOrgs.mjs:1`, `GitHubUser.mjs:1`, `IconLink.mjs:1`, `Progress.mjs:1`, `Sparkline.mjs:1`) with SCSS in `resources/scss/src/component/` and `resources/scss/theme-*/component/`.
  - **Architectural Note:** `Neo.component.Sparkline` and `Neo.component.Progress` (and `CountryFlag` / `IconLink`) are also used as general-purpose standalone components outside `Neo.grid` (for example, `apps/workstation` uses standalone `Sparkline` components in non-grid panes). Moving general-purpose components under `src/grid/column/component/` would invert the dependency direction for non-grid consumers.
- **Verdict (`op` — for `@tobiu`):** **Holds (needs scope decision: keep general-purpose components like `Sparkline` / `Progress` / `IconLink` / `CountryFlag` in `src/component/`, or only move strictly grid-specific cell components).**

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


