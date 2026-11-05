---
id: 5054
title: 'tab.header.Button: ARIA roles'
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2023-10-25T14:49:57Z'
updatedAt: '2026-11-05T10:35:25Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5054'
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
# tab.header.Button: ARIA roles

https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/tab_role

as a start, we need to add:
1. role
2. aria-selected
3. aria-controls

## Timeline

- 2023-10-25T14:49:57Z @tobiu added the `enhancement` label
- 2023-10-25T14:50:49Z @tobiu assigned to @tobiu
- 2023-10-25T15:17:04Z @tobiu referenced in commit `fe0af93` - "tab.header.Button: ARIA roles #5054 => role, aria-selected"
### @tobiu - 2023-10-25T15:18:31Z

`aria-controls` is tricky: each button needs the `tabpanel` id, which means inside the aria context the card id.

this one might not even exist so we need a TabPanel based id creation logic.

- 2023-10-30T16:15:51Z @tobiu referenced in commit `d52eeec` - "v6.9.4 (#5059)

* tab.header.Button: ARIA roles #5054 => role, aria-selected

* form.field.Picker: adjust pickerIsMounted #5056

* moving param position

* dependencies update => sass is now supporting node v21.

* controller.Base: config & method order

* controller.Base: refactoring & code cleanup

* v6.9.4"
### @github-actions - 2024-08-29T02:26:25Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:26Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:28:47Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:28:48Z @github-actions closed this issue
- 2026-10-09T16:26:55Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T19:50:03Z

#19489 set C sample · Grace · 2026-10-09: **reopened, narrowed.** Two of the three attributes shipped: tab header buttons carry `role: 'tab'` and toggle `aria-selected` (`src/tab/header/Button.mjs` L24–28, L61–63). `aria-controls` is still missing, and the header toolbar has no `tablist` role.

- 2026-10-09T20:41:23Z @neo-opus-grace cross-referenced by #5138
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:25Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. Two of the three attributes shipped. Tab header buttons carry `role: 'tab'` (`tab.header.Button` and `tab.header.EffectButton`) and toggle `aria-selected`. Still missing:
- `aria-controls` from each button to its card
- `role="tablist"` on the header toolbar
- the matching `role="tabpanel"` and `aria-labelledby` on the cards

`tablist`, `tabpanel` and `aria-controls` appear nowhere in `src/tab`.

**Verdict: holds**, with that remaining scope. Your 2023 note still applies: `aria-controls` needs the card's id, and `tab.Container` can supply it because it creates both the buttons and the cards.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


