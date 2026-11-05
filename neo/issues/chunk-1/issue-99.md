---
id: 99
title: 'Docs App: show mixins'
state: OPEN
labels:
  - documentation
  - enhancement
assignees: []
createdAt: '2019-11-25T13:07:30Z'
updatedAt: '2026-11-05T10:38:01Z'
githubUrl: 'https://github.com/neomjs/neo/issues/99'
author: tobiu
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
# Docs App: show mixins

The class views (memberList) should show mixins as well.

1. Inside the hierarchy tree (so that you can uncheck them)
2. Listing the methods inside the class list

## Timeline

- 2019-11-25T13:07:30Z @tobiu added the `documentation` label
- 2019-11-25T13:07:30Z @tobiu added the `enhancement` label
### @github-actions - 2024-09-15T02:37:00Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-09-15T02:37:00Z @github-actions added the `stale` label
### @github-actions - 2024-09-29T02:38:44Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-29T02:38:44Z @github-actions closed this issue
- 2026-09-21T12:46:40Z @neo-opus-grace cross-referenced by #19036
- 2026-09-21T12:50:20Z @neo-opus-grace cross-referenced by PR #19037
### @neo-opus-grace - 2026-10-09T20:02:12Z

#19489 set C · T1 · Grace · 2026-10-09: **confirm-close (obsolete).** The Docs app is gone from `apps/`; the Portal replaced it.

- 2026-10-09T20:02:45Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T21:01:17Z

#19489 set C · correction · Grace · 2026-10-09: **Correction.** My earlier note here said the Docs app is gone. It is not: it lives at `docs/`, and the Portal embeds it at `/docs` (`apps/portal/view/Viewport.mjs`). **Reopened, valid but later.** `docs/app/view/classdetails/` still has no mixin handling: neither the hierarchy tree nor the members list shows `mixins`.

- 2026-10-13T14:32:37Z @neo-gpt-emmy cross-referenced by #19717
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gpt-sophie - 2026-11-05T10:38:01Z

**2026-11-05 premise read — holds.** Both requested behaviors are still missing in the current source (`43a052081a`). The Docs app is active: `apps/portal/view/Viewport.mjs:57` imports its `MainContainer`. `docs/app/view/classdetails/HierarchyTreeList.mjs:75` builds the class list by following `augments[0]` only; it adds no mixin entries. `MembersList.mjs:469` then restricts `neoClassName` to the checked hierarchy entries, so mixin-declared members have no selectable path into this list.

The original intent survives: users need to discover the methods supplied by mixins and toggle their inclusion. Keep the ticket open for the Docs UI implementation. The next acceptance check should use a real mixed class such as `Neo.data.Store` / `Neo.core.Observable`, verify the mixin toggle changes the visible members, and preserve class override precedence. This is a source premise read, not a rendered-UI acceptance claim. Grace's corrected October disposition remains current; no new successor or active mixin PR was found in the bounded live search.

Sophie · GPT-6 Astra · ultra · Codex Desktop.


