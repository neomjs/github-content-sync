---
id: 44
title: 'component.DateSelector: keynav & select'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2019-11-18T01:11:11Z'
updatedAt: '2026-11-05T10:34:29Z'
githubUrl: 'https://github.com/neomjs/neo/issues/44'
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
# component.DateSelector: keynav & select

(optional) mode to not select when using the arrow keys. apply a different cls (similar the today cls) and select on enter.

config: selectOnNavigate sounds good to me.

## Timeline

- 2019-11-18T01:11:11Z @tobiu added the `enhancement` label
### @github-actions - 2024-09-15T02:37:10Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-15T02:37:11Z @github-actions added the `stale` label
### @github-actions - 2024-09-29T02:38:55Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-29T02:38:55Z @github-actions closed this issue
- 2026-09-08T07:51:41Z @neo-opus-grace cross-referenced by #18465
### @neo-opus-grace - 2026-10-09T20:00:17Z

#19489 set C · T1 · Grace · 2026-10-09: **reopened, valid but later.** `DateSelectorModel` still selects on every arrow key (`onNavKeyColumn` / `onNavKeyRow` call `select()` then `focus()`). The navigate-then-Enter mode (`selectOnNavigate`) does not exist.

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
- 2026-10-09T20:09:43Z @neo-opus-grace cross-referenced by #963
- 2026-10-15T17:01:44Z @neo-opus-ada cross-referenced by PR #19761
- 2026-11-02T18:13:46Z @neo-opus-vega cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:34:29Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `selection.DateSelectorModel` still selects on every arrow key, because `onNavKeyColumn()` and `onNavKeyRow()` both end in `me.select(id)`. The `stayInMonth` config that #47 added on 11-03 (9889010e11) changes where navigation goes, not whether it selects. There is no `selectOnNavigate`.

**Verdict: holds.** The body's design is still the shape: a navigation cls like the today cls, and select on Enter.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


