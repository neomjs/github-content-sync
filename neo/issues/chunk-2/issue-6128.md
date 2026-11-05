---
id: 6128
title: 'manager.Focus: history => scope the content to windowIds'
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2024-11-22T11:00:15Z'
updatedAt: '2026-11-05T10:34:40Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6128'
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
# manager.Focus: history => scope the content to windowIds

for the multi-window scope, it would be nice to have a separated focus history for each connected window.

![Image](https://github.com/user-attachments/assets/d0067ed4-5957-4029-ba87-b977a8bea82e)

so, the history should become an object with `windowId` props which then contain the current arrays.

closing a browser window should clear the related history.

## Timeline

- 2024-11-22T11:00:15Z @tobiu added the `enhancement` label
- 2024-11-22T11:00:15Z @tobiu added the `no auto close` label
- 2024-11-22T11:00:15Z @tobiu assigned to @tobiu
- 2026-07-10T23:09:38Z @neo-fable-clio cross-referenced by #15000
- 2026-09-04T17:53:38Z @neo-opus-ada cross-referenced by #18299
### @neo-gpt-euclid - 2026-11-05T10:34:40Z

**2026-11-05 premise read — holds; current implementation still needs the per-window boundary.** Checked Engine `6ddbed1bd5e67822510c8ab3ece3340f58994313` for the backlog sweep in #15000.

[`src/manager/Focus.mjs:21`](https://github.com/neomjs/neo/blob/6ddbed1bd5e67822510c8ab3ece3340f58994313/src/manager/Focus.mjs#L21) still declares one `history: []`; `addToHistory()` (53–57) unshifts into it without selecting a window, and `focusMove()` (87–109) reads its global first entry. An isolated call through the real singleton with `window-a`, then `window-b` produced `{isSharedArray:true,length:2,order:["window-b","window-a"]}`. No browser focus behavior is inferred from that probe.

The outcome in this issue remains valid. The implementation prescription needs to account for code added since 2024: `pendingFocusOut` and its settlement (162–190) are also singleton-wide. The eventual coherent fix should review history selection, pending focusout pairing and window-disconnect cleanup together; only replacing the array declaration would leave the pairing path shared. `test/playwright/unit/manager/Focus.spec.mjs` currently checks the single-array path. No resolving PR surfaced in the live ticket-reference search.

This is a dated intent/shape check, not a close recommendation or implementation claim. The original operator-authored body and assignment remain intact; no replacement ticket.



