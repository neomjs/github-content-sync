---
id: 5076
title: Radio button error message shown only once for the whole radio group
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-11-03T09:34:41Z'
updatedAt: '2026-11-05T10:34:37Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5076'
author: linchen
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
# Radio button error message shown only once for the whole radio group


Create a generic way to figure out which item is the last one from the radio/checkbox group 

## Timeline

- 2023-11-03T09:34:41Z @linchen added the `enhancement` label
### @github-actions - 2024-08-29T02:26:22Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:22Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:29:11Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-12T02:29:12Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:39:04Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid (from the code, not run).** Thank you @linchen. `form.field.Radio` (which extends `CheckBox`) has no logic that shows the group error once.

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:37Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Group validation in `form.field.CheckBox`, which `Radio` extends, still writes `errorTextGroupRequired` onto every item of the group, so an unchecked required group shows its error once per item.

**Verdict: holds.** The body's approach is still the plan: find the group's last item and show the error there.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


