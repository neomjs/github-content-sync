---
id: 279
title: 'component.Gallery: support for mobile'
state: OPEN
labels:
  - enhancement
  - help wanted
assignees: []
createdAt: '2020-03-14T19:57:27Z'
updatedAt: '2026-11-05T10:35:00Z'
githubUrl: 'https://github.com/neomjs/neo/issues/279'
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
# component.Gallery: support for mobile

we need to add touch events.

swipe => select the next matching item

pinch / zoom => translateZ

any other ideas?

## Timeline

- 2020-03-14T19:57:27Z @tobiu added the `enhancement` label
- 2020-03-14T19:57:27Z @tobiu added the `help wanted` label
### @github-actions - 2024-09-14T02:27:59Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-14T02:27:59Z @github-actions added the `stale` label
### @github-actions - 2024-09-28T02:32:16Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-28T02:32:17Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:00:40Z

#19489 set C · T1 · Grace · 2026-10-09: **reopened, valid but later.** `src/component/Gallery.mjs` has no touch handling at all.

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:00Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `component.Gallery` has no touch handling, and the body is an open list of ideas ("any other ideas?").

**Verdict: convert to a Discussion, together with #280**, because the touch gestures for both components are one design question. The ticket is operator-authored, so the conversion is the operator's call; it is listed on #15000.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-05T10:35:03Z @neo-opus-ada cross-referenced by #280

