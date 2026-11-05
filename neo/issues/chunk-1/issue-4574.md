---
id: 4574
title: 'table.View: createViewData() => allow renderers to return neo instances'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-07-26T12:54:54Z'
updatedAt: '2026-11-05T10:38:35Z'
githubUrl: 'https://github.com/neomjs/neo/issues/4574'
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
# table.View: createViewData() => allow renderers to return neo instances

@albert-hashani 

this would be a nice addition: if we return a neo instance, just drop the instance vdom into the cell.

we could also check for neo-configs: if the return value is an object and has a module, ntype or className => create an instance, drop in the vdom.

## Timeline

- 2023-07-26T12:54:54Z @tobiu added the `enhancement` label
### @github-actions - 2024-08-29T02:26:58Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:59Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:29:31Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:29:31Z @github-actions closed this issue
- 2026-10-09T19:52:53Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T20:38:18Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid but later.** Grid columns can host components, but `table.Body` renderers still cannot return a Neo instance.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:38:35Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - `table.View` was renamed to `Neo.table.Body` (`src/table/Body.mjs`).
  - In `src/table/Body.mjs:151-225` (`applyRendererOutput`), `column.renderer` output is handled only as a primitive or `{cls, html}` descriptor (`tag: 'td', innerHTML: rendererOutput.html`); it does not accept a `Neo.component.Base` instance (`rendererOutput.vdom`) or a Neo config (`module` / `ntype` / `className`), whereas `Neo.grid` handles component cells via `src/grid/column/Component.mjs` and `src/grid/Row.mjs:595-689`.
- **Verdict (`op`):** **Holds (target is now `src/table/Body.mjs#applyRendererOutput`)** — or operator close candidate if component cells are intentionally scoped to `Neo.grid.*`.

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


