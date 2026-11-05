---
id: 5136
title: 'toolbar.Base: Keyboard navigation & button triggers'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-12-05T10:50:21Z'
updatedAt: '2026-11-05T10:35:28Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5136'
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
# toolbar.Base: Keyboard navigation & button triggers

From an A11Y perspective, we do need keyboard navigation to move the focus to different toolbar items (buttons).

`Enter` or `Space` keys should triggers the button handler (not sure if this already happens out of the box).

Requires: https://github.com/neomjs/neo/issues/5135

## Timeline

- 2023-12-05T10:50:21Z @tobiu added the `enhancement` label
### @github-actions - 2024-08-29T02:26:08Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:09Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:28:56Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-12T02:28:56Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:39:14Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid.** `toolbar.Base` still has no keyboard navigation between its items. `main.addon.Navigator` (#5135) is the natural base.

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:27Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `toolbar.Base` still has no keyboard navigation between its items. Its three changes since 10-09 (#20046, #20081, #19983) were overflow and header-action work. `main.addon.Navigator` (#5135) exists to build on.

**Verdict: holds.**

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


