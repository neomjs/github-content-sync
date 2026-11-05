---
id: 3935
title: 'TextField & TextArea: Display character count and max length'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-01-23T12:48:52Z'
updatedAt: '2026-11-05T10:34:32Z'
githubUrl: 'https://github.com/neomjs/neo/issues/3935'
author: alberthashani
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
# TextField & TextArea: Display character count and max length

A label for character count and max length is needed in bottom right position of TextField and TextArea in this format "0 / 200" the 0 meaning the character count and 200 max length of the text.

## Timeline

- 2023-01-23T12:48:52Z @alberthashani added the `enhancement` label
### @github-actions - 2024-08-30T02:27:08Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-30T02:27:09Z @github-actions added the `stale` label
### @github-actions - 2024-09-14T02:26:07Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-14T02:26:07Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:31:19Z

#19489 set C · T5 · Grace · 2026-10-09: **reopened, valid but later.** Thank you @alberthashani. `form.field.Text` only shows `valueLength / maxLength` inside the max-length error text; there is no live "0 / 200" counter.

- 2026-10-09T20:34:19Z @neo-opus-grace cross-referenced by #19489
- 2026-10-09T20:55:09Z @neo-opus-grace cross-referenced by #5257
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:31Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `form.field.Text` still reports `valueLength / maxLength` only inside `errorTextMaxLength`, as an error, and has no live counter. Its three changes since Grace's 10-09 read (#4817, #3335, #4184) don't touch this.

**Verdict: holds.** The spec here is precise enough to build from: a "0 / 200" counter at the bottom right of TextField and TextArea.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


