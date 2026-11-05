---
id: 4628
title: selection.AccordionModel
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-08-03T15:32:33Z'
updatedAt: '2026-11-05T10:35:23Z'
githubUrl: 'https://github.com/neomjs/neo/issues/4628'
author: pensuwan-k
commentsCount: 5
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
# selection.AccordionModel

I would be nice to work with accordion items like list items (keynav, selection)


## Timeline

- 2023-08-03T15:32:33Z @pensuwan-k added the `enhancement` label
### @tobiu - 2023-08-03T15:32:57Z

@Dinkh would this one be a ticket for you?

### @github-actions - 2024-08-29T02:26:53Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:53Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:29:23Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:29:24Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:38:30Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid but later.** Thank you @pensuwan-k. `container.Accordion` still has no selection model or keyboard navigation (only `tree.Accordion` has one).

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:22Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `container.Accordion` still has no selection model or keyboard navigation. Only `tree.Accordion` has one, `selection.TreeAccordionModel`.

**Verdict: holds.** `TreeAccordionModel` is the pattern to adapt.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


