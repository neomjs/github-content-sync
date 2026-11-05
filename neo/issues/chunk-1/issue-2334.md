---
id: 2334
title: calendar.view.month.EventDragZone
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2021-06-10T12:02:47Z'
updatedAt: '2026-11-05T10:34:19Z'
githubUrl: 'https://github.com/neomjs/neo/issues/2334'
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
# calendar.view.month.EventDragZone

We need the ability to drag events into different days.

## Timeline

- 2021-06-10T12:02:47Z @tobiu added the `enhancement` label
- 2021-06-10T12:02:47Z @tobiu assigned to @tobiu
### @github-actions - 2024-09-01T02:38:43Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-01T02:38:43Z @github-actions added the `stale` label
### @github-actions - 2024-09-16T02:37:00Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-16T02:37:00Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:15:46Z

#19489 set C · T3 · Grace · 2026-10-09: **reopened, valid but later.** Only the week view can drag events (`src/calendar/view/week/EventDragZone.mjs`); the month view has no event drag zone.

- 2026-10-09T20:18:24Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:19Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `src/calendar/view/month/` holds only `Component.mjs`. Event dragging exists for the week view alone (`week/EventDragZone.mjs`, `week/plugin/DragDrop.mjs`).

**Verdict: holds.** The week view's drag zone is the pattern to port.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


