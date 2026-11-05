---
id: 3782
title: HighlightJS resources
state: OPEN
labels:
  - bug
assignees: []
createdAt: '2023-01-04T20:39:38Z'
updatedAt: '2026-11-05T10:35:12Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3782'
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
# HighlightJS resources

right now, we have the resources twice inside the repo, which is not good for maintenance and package file size.

the current approach breaks inside the dist modes:
<img width="1791" alt="Screenshot 2023-01-04 at 21 28 48" src="https://user-images.githubusercontent.com/1177434/210645513-61328a35-0134-426f-963d-6b31c7444010.png">

since the top level resources folder does get copied into dist already, we should move the files there.

@Dinkh 

## Timeline

- 2023-01-04T20:39:38Z @tobiu added the `bug` label
### @github-actions - 2024-08-30T02:27:17Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:27:17Z @github-actions added the `stale` label
### @github-actions - 2024-09-14T02:26:17Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-14T02:26:17Z @github-actions closed this issue
- 2026-10-09T19:52:53Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T20:31:10Z

#19489 set C · T5 · Grace · 2026-10-09: **reopened, valid.** The highlight.js bundle and its two custom themes still exist twice: `docs/resources/lib/` and `examples/component/toast/resources/lib/`.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:12Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Both highlight.js copies still exist: `docs/resources/lib/highlight/` and `examples/component/toast/resources/lib/highlight/`.

**Verdict: holds, as a defect.** The body's fix still applies: keep one copy under the top-level `resources/`, which dist already copies.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


