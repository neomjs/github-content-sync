---
id: 5219
title: 'form.field.Date: Accessibility enhancements'
state: OPEN
labels:
  - enhancement
assignees:
  - ExtAnimal
createdAt: '2024-02-13T08:35:50Z'
updatedAt: '2026-11-05T10:34:42Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5219'
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
# form.field.Date: Accessibility enhancements

there are quite some official specs for this widget, which we should honor.

related content (example link):
https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/examples/datepicker-dialog/


## Timeline

- 2024-02-13T08:35:50Z @tobiu added the `enhancement` label
- 2024-02-13T08:35:50Z @tobiu assigned to @ExtAnimal
- 2024-02-13T08:37:02Z @tobiu cross-referenced by #5217
- 2024-02-14T10:22:18Z @tobiu cross-referenced by #5223
### @github-actions - 2024-08-29T02:25:45Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:25:45Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:28:25Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-12T02:28:26Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:53:00Z

#19489 set C · T7 · Grace · 2026-10-09: **reopened, valid but later.** `component.DateSelector` still carries no ARIA roles or attributes (compare the WAI-ARIA datepicker dialog pattern).

- 2026-10-09T21:02:53Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:42Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Neither `component.DateSelector` nor `form.field.Date` carries a `role` or `aria-*` attribute: zero matches in either file.

**Verdict: holds.** The WAI-ARIA date picker dialog pattern linked in the body is the spec.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


