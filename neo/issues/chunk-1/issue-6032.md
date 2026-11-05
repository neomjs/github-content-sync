---
id: 6032
title: 'button.Base: editRoute config => change the default value to false'
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2024-10-15T22:41:48Z'
updatedAt: '2026-11-05T10:35:48Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6032'
author: tobiu
commentsCount: 2
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
# button.Base: editRoute config => change the default value to false

i think the only the Covid & SharedCovid apps are still using parameterized routes. the default value should be false (more common purpose).

i need to check all affected examples & apps for this change, to ensure there are no side effects.


## Timeline

- 2024-10-15T22:41:48Z @tobiu added the `enhancement` label
- 2024-10-15T22:41:48Z @tobiu assigned to @tobiu
### @github-actions - 2025-01-14T02:26:58Z

This issue is stale because it has been open for 90 days with no activity.

- 2025-01-14T02:26:59Z @github-actions added the `stale` label
- 2025-01-25T22:34:55Z @tobiu removed the `stale` label
- 2025-01-25T22:34:55Z @tobiu added the `no auto close` label
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-opus-ada - 2026-11-05T10:35:48Z

**Premise read, 2026-11-05** (backlog sweep #15000), at dev `6ddbed1bd5`. `button.Base#editRoute` still defaults to `true`. `editRoute` appears in `apps/covid`, `apps/sharedcovid`, `apps/portal/Overwrites.mjs` and `examples/serverside/toolbarItems/Overwrites.mjs`.

**Verdict: holds.** The remaining work is the audit the body asks for. Every button with a `route` that relies on the `true` default needs an explicit `editRoute: true` before the default flips.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


