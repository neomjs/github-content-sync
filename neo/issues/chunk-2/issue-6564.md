---
id: 6564
title: 'grid.plugin.AnimateRows: store listeners improvement'
state: OPEN
labels:
  - enhancement
  - no auto close
assignees:
  - tobiu
createdAt: '2025-03-09T22:49:19Z'
updatedAt: '2026-11-05T10:38:43Z'
githubUrl: 'https://github.com/neomjs/neo/issues/6564'
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
# grid.plugin.AnimateRows: store listeners improvement

* Instead of rebinding view based store listeners logic, the cleaner way would be to remove the relevant view based listeners and add own listeners to the store
* This way, we could easily restore the view to its previous state, in case the plugin needs to get disabled at run-time

## Timeline

- 2025-03-09T22:49:19Z @tobiu added the `enhancement` label
- 2025-03-09T22:49:19Z @tobiu added the `no auto close` label
- 2025-03-09T22:49:19Z @tobiu assigned to @tobiu
- 2026-06-14T10:56:51Z @neo-gpt-euclid cross-referenced by #13219
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-maryam - 2026-11-05T10:38:42Z

### Backlog Premise-Read (`neomjs/neo#15000`, `2026-11-05`)

- **Current Code Reality:**
  - In `src/grid/plugin/AnimateRows.mjs:93-102` (`construct`), the plugin still swaps `owner.onStoreFilter` and `owner.onStoreLoad` via `owner.store = null; me.#bodyHandlers = {filter: owner.onStoreFilter, load: owner.onStoreLoad}; ... owner.store = store;`.
  - PR #19774 (`2d76f3f6df` for #6858) added `me.#bodyHandlers` (`line 66`) and `standAside()` (`lines 273-286`) so the plugin dynamically hands `filter` and `load` events back to `me.#bodyHandlers` whenever `contentSizedRows` (`!!owner.rowGeometry`) is active (`lines 196, 210`).
  - However, `destroy()` (`src/grid/plugin/AnimateRows.mjs:146-164`) does not yet restore `owner.onStoreFilter = me.#bodyHandlers.filter` / `owner.onStoreLoad = me.#bodyHandlers.load` (or use independent `store.on(...)` listeners) if the plugin is destroyed or disabled while the grid body remains alive.
- **Verdict (`op` — for `@tobiu`):** **Holds (partially addressed by `#bodyHandlers` + `standAside()` in PR `#19774`, with listener restoration on disable/destroy still applicable at `src/grid/plugin/AnimateRows.mjs:93-102, 146-164`)** — or close candidate if `standAside()` satisfies the runtime disable requirement.

— Maryam (Gemini 4 Argon, Antigravity) · session 70151411-91ec-4496-b8f0-ea3b31ddb60b 🧭


