---
id: 291
title: 'component.Helix: Loading Spinner'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2020-03-16T08:38:07Z'
updatedAt: '2026-11-05T10:35:06Z'
githubUrl: 'https://github.com/neomjs/neo/issues/291'
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
# component.Helix: Loading Spinner

visual feedback when loading the matching store.

connect to beforeload & load events, in case the component is mounted & not hidden, show the spinner.

## Timeline

- 2020-03-16T08:38:08Z @tobiu added the `enhancement` label
### @github-actions - 2024-09-14T02:27:54Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-14T02:27:55Z @github-actions added the `stale` label
### @github-actions - 2024-09-28T02:32:10Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-28T02:32:10Z @github-actions closed this issue
- 2026-08-31T19:08:02Z @neo-gpt-emmy cross-referenced by #17416
### @neo-opus-grace - 2026-10-09T20:00:54Z

#19489 set C · T1 · Grace · 2026-10-09: **reopened, valid but later.** Components have a generic `isLoading_` (`src/component/Base.mjs`), but `Helix` does not connect its store's load events to it.

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
- 2026-11-05T10:35:05Z @neo-opus-ada cross-referenced by #290
### @neo-opus-ada - 2026-11-05T10:35:06Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `component.Base` has `isLoading_`, but `component.Helix` still doesn't connect its store's `beforeLoad` and `load` events to it; the file is unchanged since 10-09.

**Verdict: holds.** It is small, because the body's plan maps onto the existing `isLoading_` config. #290 is the same change for the Gallery.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


