---
id: 6858
title: 'grid.plugin.AnimateRows: updateView() => row ids'
state: OPEN
labels:
  - bug
  - no auto close
assignees:
  - tobiu
createdAt: '2025-06-23T11:16:12Z'
updatedAt: '2026-10-12T18:16:22Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6858'
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
# grid.plugin.AnimateRows: updateView() => row ids

* Since row ids are now index based, we need a different mechanism. The internal `map` should store record ids instead.

## Timeline

- 2025-06-23T11:16:12Z @tobiu assigned to @tobiu
- 2025-06-23T11:16:14Z @tobiu added the `bug` label
### @github-actions - 2025-09-22T02:47:59Z

This issue is stale because it has been open for 90 days with no activity.

- 2025-09-22T02:48:00Z @github-actions added the `stale` label
- 2025-09-22T08:34:12Z @tobiu removed the `stale` label
- 2025-09-22T08:34:12Z @tobiu added the `no auto close` label
- 2026-06-14T10:56:51Z @neo-gpt cross-referenced by #13219
- 2026-10-12T18:16:20Z @neo-opus-ada cross-referenced by #19665
### @neo-opus-ada - 2026-10-12T18:16:22Z

Current state, measured at `017c2a6773` in headless Chromium on `examples/grid/animatedRowSorting`: sorting by a header click and filtering via the Firstname field each throw `owner.createRow is not a function`. The pooled `grid.Body` has no `createRow`. The plugin also maps rows by vdom `id`, while `Body#getVdomRoot().cn` now holds Row component references (`createVdomReference()`), so the move/insert bookkeeping in `updateView()` has no match. A repair is a rewrite onto the Row pool (record-id keyed, as this ticket says), not a patch. No spec covers the plugin.

Content-sized grids (#19658) take the plugin out of play instead: #19665.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-12T18:32:43Z @neo-opus-ada cross-referenced by PR #19677
- 2026-10-12T19:53:53Z @tobiu referenced in commit `358af02` - "feat(grid): AnimateRows stands aside in content-sized mode (#19665) (#19677)

* feat(grid): AnimateRows stands aside in content-sized mode (#19665)

The plugin animates rows to fixed-height positions and does not read the row geometry. With
contentSizedRows it now takes its row transitions off the body, warns once that animatedRowSorting is
ignored, and hands the store's filter and load events back to the body's own handlers, so rows render at
their geometry offsets. This works whichever comes first, the mode or the plugin.

Its fixed-height path is untouched. That path throws `owner.createRow is not a function` since the
pooled Body; its repair is #6858.

* test(grid): AnimateRows stands aside in both activation orders and when declared by hand (#19665)

* fix(grid): AnimateRows follows the content-sized mode as it switches, before rows re-render (#19665)

grid.View#updateRowGeometry now has each body fire `rowGeometryChange` before a mode switch re-renders
every row. AnimateRows listens on its body: switching content-sized rows on takes its row transitions
off and warns, before measured heights move any row, and switching back restores them, with no store
event needed. The plugin releases the listener when it is destroyed."

