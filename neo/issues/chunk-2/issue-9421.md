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
updatedAt: '2026-11-06T13:42:44Z'
githubUrl: 'https://github.com/neomjs/neo/issues/9421'
author: tobiu
commentsCount: 3
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

- 2026-03-09T16:23:08Z @tobiu assigned to @tobiu
- 2026-03-09T16:23:09Z @tobiu added the `refactoring` label
- 2026-03-09T16:23:10Z @tobiu added the `architecture` label
- 2026-03-09T16:23:10Z @tobiu added the `grid` label
### @github-actions - 2026-06-08T06:11:46Z

This issue is stale because it has been open for 90 days with no activity.

- 2026-06-08T06:11:47Z @github-actions added the `stale` label
- 2026-06-16T17:03:56Z @tobiu removed the `stale` label
- 2026-06-16T17:03:56Z @tobiu added the `no auto close` label
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:41:32Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - `src/grid/column/component/Tree.mjs` (`Neo.grid.column.component.Tree`) already lives under `src/grid/column/component/`.
  - The 6 components listed in `#9421` (`CountryFlag.mjs`, `GitHubOrgs.mjs`, `GitHubUser.mjs`, `IconLink.mjs`, `Progress.mjs`, `Sparkline.mjs`) still reside in `src/component/` (imported by `src/grid/column/CountryFlag.mjs:1`, `GitHubOrgs.mjs:1`, `GitHubUser.mjs:1`, `IconLink.mjs:1`, `Progress.mjs:1`, `Sparkline.mjs:1`) with SCSS in `resources/scss/src/component/` and `resources/scss/theme-*/component/`.
  - **Architectural Note:** `Neo.component.Sparkline` and `Neo.component.Progress` (and `CountryFlag` / `IconLink`) are also used as general-purpose standalone components outside `Neo.grid` (for example, `apps/workstation` uses standalone `Sparkline` components in non-grid panes). Moving general-purpose components under `src/grid/column/component/` would invert the dependency direction for non-grid consumers.
- **Verdict (`op` — for `@tobiu`):** **Holds (needs scope decision: keep general-purpose components like `Sparkline` / `Progress` / `IconLink` / `CountryFlag` in `src/component/`, or only move strictly grid-specific cell components).**

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭

### @neo-opus-grace - 2026-11-06T13:42:44Z

**Correction to the 2026-11-05 premise read above: the intent holds, so this is a delivery lane, not a scope decision.** (Backlog householding #15000, checked at `dev` 28fc096649.)

The read said Workstation uses `Sparkline` as a standalone component outside grids. It does not. `apps/workstation/view/FeedPane.mjs:52` and `ScalePane.mjs:67` use `type: 'sparkline'`, the grid column type, so the component renders inside grid cells there too. Importers of each component, outside its own `src/grid/column/` wrapper:

| Component | Importers outside `src/grid/column/` |
|---|---|
| `Sparkline`, `IconLink`, `CountryFlag`, `GitHubOrgs`, `GitHubUser` | none |
| `Progress` | `examples/component/progress/MainContainer.mjs` (the standalone example) |

So five of the six are grid-cell-only, as the ticket assumed. `Progress` is also a standalone component with its own example: it either stays in `src/component/` or moves along with an updated example.

What delivery has to handle:
- Each move changes a `className` (`Neo.component.*` → `Neo.grid.column.component.*`) and an import path. Code that imports the old path (e.g. `devindex`, which builds on the published engine) needs the change in its release notes.
- SCSS moves for every theme, plus the `src/grid/column/*` imports.

🖖 Grace (Opus 5.5, Claude Code) · session bc1d980b-3b06-4392-985e-dc05b2946e87

- 2026-11-06T14:57:15Z @neo-opus-grace cross-referenced by PR #20435

