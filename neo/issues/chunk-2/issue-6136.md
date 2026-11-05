---
id: 6136
title: form.field.Select
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2024-12-05T19:02:44Z'
updatedAt: '2026-11-05T10:34:45Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6136'
author: tobiu
commentsCount: 1
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
# form.field.Select

A ComboBox supports typing into an input field, while a default `select` tag only supports selecting an `option`.

This topic gets interesting, since there is a stage 2 proposal => already implemented in Chrome, but hidden inside an experimental feature: https://developer.chrome.com/blog/rfc-customizable-select

We should create the base class and make it configurable & extensible for custom option renderers.

It should extend `form.field.Base` (and not `form.field.Text`). It should use a store, but also work with passing an options array (creating a default store as needed).

Scope: neo v9

## Timeline

- 2024-12-05T19:02:44Z @tobiu added the `enhancement` label
- 2024-12-05T19:02:44Z @tobiu added the `no auto close` label
- 2024-12-05T19:02:44Z @tobiu assigned to @tobiu
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:44Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. There is still no `form.field.Select`; `form.field.ComboBox` is the only picker field. The idea stands, but two lines in the body date from 2024: "Scope: neo v9", and the experimental status of the customizable-select proposal it cites. Re-check that status before pickup.

**Verdict: rewrite** the scope and platform lines; the ticket holds after that. The ticket is operator-authored, so the rewrite is the operator's call; it is listed on #15000.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


