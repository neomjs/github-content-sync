---
id: 5086
title: buildScripts/moveFileOrFolder
state: OPEN
labels:
  - enhancement
assignees:
  - tobiu
createdAt: '2023-11-07T14:25:33Z'
updatedAt: '2026-11-05T10:58:23Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5086'
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
# buildScripts/moveFileOrFolder

refactoring is kind of painful when using WebStorm or VSCode.

how it should work:
1. adjust the import paths (and keep them relative)
2. update the className (3 spots), matching to the new file / folder path
3. in case component based files get moved, also move their related scss files (src & themes)
4. honor the framework & workspace namespaces as targets
5. update all files which are using moved files (imports)

## Timeline

- 2023-11-07T14:25:33Z @tobiu added the `enhancement` label
- 2023-11-07T14:25:33Z @tobiu assigned to @tobiu
### @github-actions - 2024-08-29T02:26:20Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:20Z @github-actions added the `stale` label
### @github-actions - 2024-09-12T02:29:08Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-12T02:29:09Z @github-actions closed this issue
- 2026-10-09T16:26:55Z @neo-opus-grace cross-referenced by #19489
### @neo-opus-grace - 2026-10-09T19:50:12Z

#19489 set C sample · Grace · 2026-10-09: **reopened, valid but later.** `buildScripts` still has no move script (only the `create/*` generators), so moving a class still means editing its imports, `className` and SCSS files by hand.

- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-julia - 2026-11-05T10:58:23Z

### Premise read & verdict (`neomjs/neo#15000` · 2026-11-05 · `@neo-gemini-julia`)

- **Verdict:** **holds (operator-authored)**
- **Premise check against current source (`dev@14d6326fc1`):**
  - `buildScripts/` has `create/*` generators (`class.mjs`, `component.mjs`, `scss.mjs`, etc.) but still has no `moveFileOrFolder.mjs` refactoring script to atomically update relative ES module imports, the 3 `className` occurrences (`@class` JSDoc tag, `static config.className`, `Neo.setupClass`), companion SCSS files across `resources/scss/src/` and `resources/scss/theme-*/`, and inbound consumer imports.
- **Recommendation:** Premise still holds; leaving open for `@tobiu`.

🏛️ Julia (`@neo-gemini-julia`)


