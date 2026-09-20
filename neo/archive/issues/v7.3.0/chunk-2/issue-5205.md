---
id: 5205
title: 'NEO.main.DomAccesss: monitorAutoGrowHandler -> too much space at bottom'
state: CLOSED
labels:
  - bug
  - stale
assignees:
  - ExtAnimal
createdAt: '2024-02-08T13:00:18Z'
updatedAt: '2024-09-12T02:28:34Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5205'
author: pensuwan-k
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 2
  signals: []
blockedBy: []
blocking: []
closedAt: '2024-09-12T02:28:33Z'
---
# NEO.main.DomAccesss: monitorAutoGrowHandler -> too much space at bottom

When rendering textbox with a lot of content, scrollbar get shown initially, so the scrollHeight adds one more line.
!Screenshot 2024-02-08 at 13 56 07 [QUARANTINED_URL: github.com]
!Screenshot 2024-02-08 at 13 56 26 [QUARANTINED_URL: github.com]



## Timeline

- 2024-02-08T13:00:18Z @pensuwan-k added the `bug` label
- 2024-02-08T13:00:46Z @tobiu assigned to @ExtAnimal
### @github-actions - 2024-08-29T02:25:50Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:25:50Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:28:33Z

This issue was closed because it has been inactive for 14 days since being marked as stale.


