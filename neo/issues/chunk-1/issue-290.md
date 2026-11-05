---
id: 290
title: 'component.Gallery: Loading Spinner'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2020-03-16T08:37:41Z'
updatedAt: '2026-11-05T10:35:05Z'
githubUrl: 'https://github.com/neomjs/neo/issues/290'
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
# component.Gallery: Loading Spinner

visual feedback when loading the matching store.

connect to beforeload & load events, in case the component is mounted & not hidden, show the spinner.

## Timeline

- 2020-03-16T08:37:41Z @tobiu added the `enhancement` label
### @github-actions - 2024-09-14T02:27:55Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-14T02:27:56Z @github-actions added the `stale` label
### @github-actions - 2024-09-28T02:32:11Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-28T02:32:12Z @github-actions closed this issue
- 2026-08-31T19:08:02Z @neo-gpt-emmy cross-referenced by #17416
- 2026-08-31T20:35:45Z @neo-opus-grace cross-referenced by #17920
### @neo-opus-grace - 2026-10-09T20:00:49Z

#19489 set C · T1 · Grace · 2026-10-09: **reopened, valid but later.** Components have a generic `isLoading_` (`src/component/Base.mjs`), but `Gallery` does not connect its store's load events to it.

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:04Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `component.Base` has `isLoading_`, but `component.Gallery` still doesn't connect its store's `beforeLoad` and `load` events to it; the file is unchanged since 10-09.

**Verdict: holds.** It is small, because the body's plan maps onto the existing `isLoading_` config. #291 is the same change for the Helix.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-05T10:35:07Z @neo-opus-ada cross-referenced by #291

