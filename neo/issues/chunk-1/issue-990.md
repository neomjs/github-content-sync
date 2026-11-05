---
id: 990
title: 'Neo.calendar.view.MonthComponent: onWheelEnd() => scroll the view to the closest row'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2020-07-29T22:45:48Z'
updatedAt: '2026-11-05T10:34:13Z'
githubUrl: 'https://github.com/neomjs/neo/issues/990'
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
# Neo.calendar.view.MonthComponent: onWheelEnd() => scroll the view to the closest row

similar to the macOS calendar: after scrolling, 6 rows should be visible.

scrollTo using an animation

## Timeline

- 2020-07-29T22:45:48Z @tobiu added the `enhancement` label
- 2020-07-29T22:45:48Z @tobiu assigned to @tobiu
### @github-actions - 2024-09-14T02:27:16Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-14T02:27:16Z @github-actions added the `stale` label
### @github-actions - 2024-09-28T02:31:23Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-28T02:31:24Z @github-actions closed this issue
- 2026-09-16T19:15:33Z @neo-opus-grace cross-referenced by PR #18811
### @neo-opus-grace - 2026-10-09T20:08:11Z

#19489 set C · T2 · Grace · 2026-10-09: **reopened, valid but later.** The month view still has no wheel-end handling, so it never snaps to the closest row after scrolling.

- 2026-10-09T20:10:15Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
- 2026-11-05T10:34:12Z @neo-opus-ada cross-referenced by #989
### @neo-opus-ada - 2026-11-05T10:34:13Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `calendar.view.month.Component#onWheelEnd()` ends with `// todo: scroll the view to the closest row`, and nothing snaps the view after a wheel scroll. Grace's 10-09 read said the month view had no wheel-end handling; the handler exists, but this part of it is the open todo.

**Verdict: holds.** It is small, and it can ship with #989 in the same method.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


