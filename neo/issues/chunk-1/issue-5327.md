---
id: 5327
title: 'list.Base: useCheckBoxes_ => make it possible to link the checked state to a record field'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2024-03-13T13:22:09Z'
updatedAt: '2026-11-05T10:36:11Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5327'
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
# list.Base: useCheckBoxes_ => make it possible to link the checked state to a record field

- right now the checked state is limited to selections
- it should optionally be possible to link it to record fields
- an item selection needs to update the checked state inside the record then

@maxrahder 

## Timeline

- 2024-03-13T13:22:09Z @tobiu added the `enhancement` label
- 2024-03-13T13:22:10Z @tobiu assigned to @tobiu
### @github-actions - 2024-08-29T02:25:28Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:25:29Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:28:03Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-12T02:28:04Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:53:12Z

#19489 set C · T7 · Grace · 2026-10-09: **reopened, valid but later.** `list.Base#useCheckBoxes_` only toggles a CSS class, so the check icons follow the selection; they cannot link to a record field.

- 2026-10-09T21:02:53Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:36:11Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `list.Base#useCheckBoxes_` still only toggles a CSS class, so the check icons follow the selection, and nothing links them to a record field. The `list.Base` changes since 10-09 were search highlighting and record re-rendering.

**Verdict: holds.**

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


