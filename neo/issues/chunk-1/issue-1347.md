---
id: 1347
title: 'table.header.Toolbar: switch to the new toolbar.SortZone'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2020-11-01T22:42:37Z'
updatedAt: '2026-11-05T10:38:29Z'
githubUrl: 'https://github.com/neomjs/neo/issues/1347'
author: tobiu
commentsCount: 4
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
# table.header.Toolbar: switch to the new toolbar.SortZone

we already have column re-sorting via drag&drop in place, but it would be a lot nicer to have it work the same way as re-sorting tabs.

## Timeline

- 2020-11-01T22:42:37Z @tobiu added the `enhancement` label
- 2020-11-01T22:42:37Z @tobiu assigned to @tobiu
### @github-actions - 2024-09-13T02:30:57Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-13T02:30:58Z @github-actions added the `stale` label
### @github-actions - 2024-09-27T02:34:09Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-27T02:34:09Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:08:30Z

#19489 set C · T2 · Grace · 2026-10-09: **reopened, valid but later.** The grid's header reorders columns through a SortZone (`src/grid/header/Toolbar.mjs`); the table's header (`src/table/header/Toolbar.mjs`) still does not.

- 2026-10-09T20:10:15Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:38:29Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - `src/grid/header/Toolbar.mjs:221-244` implements `afterSetDragResortable(value, oldValue)`, lazy-loading `src/draggable/grid/header/toolbar/SortZone.mjs` (which extends `Neo.draggable.toolbar.SortZone`).
  - `src/table/header/Toolbar.mjs:27` declares `dragResortable: true` in `static config`, but `src/table/header/Toolbar.mjs` (`lines 1–212`) has no `afterSetDragResortable` hook or `SortZone` integration.
- **Verdict (`op`):** **Holds** (valid premise at `src/table/header/Toolbar.mjs:27`, low priority relative to `Neo.grid.*`; or operator close candidate if `Neo.table.*` feature parity is superseded by `Neo.grid.*`).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


