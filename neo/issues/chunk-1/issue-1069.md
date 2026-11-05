---
id: 1069
title: 'calendar.view.WeekComponent: trigger the infinite scrolling onDragScroll (edge)'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2020-08-14T20:49:59Z'
updatedAt: '2026-11-05T10:34:17Z'
githubUrl: 'https://github.com/neomjs/neo/issues/1069'
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
# calendar.view.WeekComponent: trigger the infinite scrolling onDragScroll (edge)

we do need a global scroll event listener.

drag&scroll will trigger scroll events, so in case we reach an edge, trigger a "paging" op.

## Timeline

- 2020-08-14T20:49:59Z @tobiu added the `enhancement` label
- 2020-08-14T20:52:02Z @tobiu assigned to @tobiu
### @github-actions - 2024-09-13T02:31:22Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-13T02:31:22Z @github-actions added the `stale` label
### @github-actions - 2024-09-27T02:34:34Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-27T02:34:34Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:08:25Z

#19489 set C · T2 · Grace · 2026-10-09: **reopened, valid but later.** `calendar.view.week.Component` has no scroll-edge paging, so drag-scrolling to an edge does not page.

- 2026-10-09T20:10:15Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:17Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `calendar.view.week.Component#onWheel` pages columns in and out only on a horizontal wheel delta. The event drag zone scrolls the same container (`week/plugin/DragDrop.mjs` hands it `scrollContainerId`) without triggering that paging, so I expect a drag held at the edge to stop at the buffered columns. That expectation comes from the code; I did not run it.

**Verdict: holds.**

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


