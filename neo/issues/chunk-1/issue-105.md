---
id: 105
title: 'component.Helix: buffered sorting'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2019-11-26T12:41:27Z'
updatedAt: '2026-11-05T10:34:55Z'
githubUrl: 'https://github.com/neomjs/neo/issues/105'
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
# component.Helix: buffered sorting

Now that component.Helix does support a maxItems config, sorting will cause bugs.

The initial sorting implementation was meant to sort the items which are visible inside the view.

With more items inside the store than inside the view, a sorting operation needs to remove or add items as needed.

## Timeline

- 2019-11-26T12:41:27Z @tobiu added the `enhancement` label
- 2019-11-28T11:24:01Z @tobiu referenced in commit `3559671` - "component.Helix: buffered sorting #105 => disabled the sorting buttons until the buffered sorting is in place"
### @github-actions - 2024-09-15T02:36:58Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-15T02:36:59Z @github-actions added the `stale` label
### @github-actions - 2024-09-29T02:38:42Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-29T02:38:43Z @github-actions closed this issue
- 2026-09-04T19:54:30Z @neo-opus-grace cross-referenced by #18331
### @neo-opus-grace - 2026-10-09T20:00:26Z

#19489 set C · T1 · Grace · 2026-10-09: **reopened, valid (from the code, not run).** `Helix#onSort` → `sortItems()` → `refresh()` transforms the first `min(maxItems, count)` records, but nothing adds or removes items when a sort moves records across the `maxItems` boundary (`src/component/Helix.mjs`, `maxItems_` default 300).

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:55Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `component.Helix` is unchanged since Grace's 10-09 read. A sort transforms the first `min(maxItems, count)` records, and nothing adds or removes items when a sort moves records across the `maxItems` boundary. I read this from the code; I did not run it.

**Verdict: holds, as a defect.** Sorting a store larger than `maxItems` keeps the wrong items in view.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


