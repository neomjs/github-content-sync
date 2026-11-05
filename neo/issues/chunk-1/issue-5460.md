---
id: 5460
title: 'plugin.Responsive: owner.on({mounted: ''onOwnerMounted''})'
state: OPEN
labels:
  - enhancement
assignees:
  - Dinkh
createdAt: '2024-06-23T08:31:45Z'
updatedAt: '2026-11-05T10:35:43Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5460'
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
# plugin.Responsive: owner.on({mounted: 'onOwnerMounted'})

we need to move the resize listener logic outside of `construct()`.

rationale: moving a component into the viewport of a different window => we still want it to be responsive.

## Timeline

- 2024-06-23T08:31:45Z @tobiu added the `enhancement` label
- 2024-06-23T08:31:45Z @tobiu assigned to @Dinkh
- 2024-06-23T08:31:58Z @tobiu changed title from **plugin.Responsive: owner.on({mounted: 'onOwnerMounted'}** to **plugin.Responsive: owner.on({mounted: 'onOwnerMounted'})**
### @github-actions - 2024-09-22T02:36:40Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-22T02:36:41Z @github-actions added the `stale` label
### @github-actions - 2024-10-06T02:38:06Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-10-06T02:38:07Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:53:29Z

#19489 set C · T7 · Grace · 2026-10-09: **reopened, valid.** The `plugin.Responsive` cleanup (#5458–#5461) is still open: its resize listener is still set up at construction, not when the owner mounts.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
- 2026-11-05T10:35:41Z @neo-opus-ada cross-referenced by #5459
### @neo-opus-ada - 2026-11-05T10:35:43Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `plugin.Responsive` still sets up its resize listener during construction, not when the owner mounts, so a component moved into another window's viewport keeps listening to the old one. The plugin is unchanged since 10-09.

**Verdict: holds.** It belongs in one bundle with #5458, #5459 and #5461; see #5458.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-05T10:35:46Z @neo-opus-ada cross-referenced by #5461

