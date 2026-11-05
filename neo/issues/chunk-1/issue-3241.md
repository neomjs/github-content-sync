---
id: 3241
title: 'table.header.Toolbar: create support for resizing table columns via drag&drop'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2022-07-01T08:17:41Z'
updatedAt: '2026-11-05T10:38:32Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3241'
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
# table.header.Toolbar: create support for resizing table columns via drag&drop

*(No description provided)*

## Timeline

- 2022-07-01T08:17:41Z @tobiu added the `enhancement` label
- 2022-07-01T08:17:41Z @tobiu assigned to @tobiu
### @github-actions - 2024-08-30T02:28:06Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:28:06Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:30:18Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:30:18Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:23:32Z

#19489 set C · T4 · Grace · 2026-10-09: **reopened, valid but later.** The grid's header resizes columns (`header/plugin/Resizable.mjs`); the table's header (`src/table/header`) still cannot.

- 2026-10-09T20:26:18Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:38:32Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - `src/grid/header/Toolbar.mjs:184-205` implements `afterSetResizable(value, oldValue)`, lazy-loading `src/grid/header/plugin/Resizable.mjs`.
  - `src/table/header/Toolbar.mjs` (`lines 1–212`) has no `resizable_` config and no `plugin/Resizable.mjs` counterpart under `src/table/header/`.
- **Verdict (`op`):** **Holds** (valid premise at `src/table/header/Toolbar.mjs:1-212`, low priority relative to `Neo.grid.*`; or operator close candidate if `Neo.table.*` feature parity is superseded by `Neo.grid.*`).

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


