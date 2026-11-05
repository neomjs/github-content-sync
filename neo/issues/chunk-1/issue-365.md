---
id: 365
title: 'Covid.view.MainContainerController: onHashChange() refactoring'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2020-03-24T19:35:32Z'
updatedAt: '2026-11-05T10:52:43Z'
githubUrl: 'https://github.com/neomjs/neo/issues/365'
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
# Covid.view.MainContainerController: onHashChange() refactoring

there is a check in place if data got loaded and if not use a setTimeout() call.

this was just a quick hack.

onHashChange needs to store the params of its last call inside the class.

in case the api is loading or there is no data available => do nothing.

instead, when data got loaded, trigger onHashChange afterwards. Best with a 50-100ms delay to ensure the data got mounted into the real dom already.

## Timeline

- 2020-03-24T19:35:32Z @tobiu added the `enhancement` label
### @github-actions - 2024-09-14T02:27:44Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-14T02:27:44Z @github-actions added the `stale` label
### @github-actions - 2024-09-28T02:31:56Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-28T02:31:57Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:07:59Z

#19489 set C · T2 · Grace · 2026-10-09: **reopened, valid but later.** `Covid.view.MainContainerController#onHashChange` still retries itself via `me.timeout(10)` until the view exists, the quick fix this ticket wanted replaced.

- 2026-10-09T20:10:15Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:52:43Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `Covid.view.MainContainerController#onHashChange` still retries itself with `me.timeout(10)` until the view exists. That is the quick hack this ticket wants replaced.

**Verdict: holds.** The body's plan still fits: store the last call's params, and re-run once the data has loaded.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


