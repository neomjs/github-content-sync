---
id: 5007
title: 'config order: add an exception for `layout` to always be in front of `items`'
state: OPEN
labels:
  - enhancement
  - help wanted
  - good first issue
assignees: []
createdAt: '2023-10-12T05:42:21Z'
updatedAt: '2026-11-05T10:30:54Z'
githubUrl: 'https://github.com/neomjs/neo/issues/5007'
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
# config order: add an exception for `layout` to always be in front of `items`

@Dinkh had the idea and it makes sense to me.

we need to add the new rule here:
https://github.com/neomjs/neo/blob/dev/.github/CODING_GUIDELINES.md

and then adjust all view related files inside the repo. my guess is 100+ spots.

should be an very easy, but time consuming task.

## Timeline

- 2023-10-12T05:42:21Z @tobiu added the `enhancement` label
- 2023-10-12T05:42:21Z @tobiu added the `help wanted` label
- 2023-10-12T05:42:21Z @tobiu added the `good first issue` label
### @github-actions - 2024-08-29T02:26:34Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:35Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:28:59Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:29:00Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:38:59Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, valid.** `.github/CODING_GUIDELINES.md` still has no layout-before-items exception; #5402 fixes the same section's ordering wording.

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-julia - 2026-11-05T10:30:54Z

### Backlog Premise Read (`2026-11-05` · `neomjs/neo#15000`)

- **Verdict:** **holds / operator decision (`@tobiu`)**
- **Premise check against `dev`:**
  - In `.github/CODING_GUIDELINES.md`:
    - **Rule 21 (`lines 287-294`, `static config` order):** Specifies `className` first, `ntype` second, all other configs ordered alphabetically, `_vdom` last. In `src/container/Base.mjs:102,113`, `items_` (`line 102`) currently precedes `layout_` (`line 113`) alphabetically.
    - **Rule 28 (`lines 379-385`, config objects):** Already includes the single-line vs. multi-line blank-line reset exception (*"You can also sort everything which can get described in 1 line alphabetically and use an empty blank line afterwards. This 'resets' the block formatting and order. Afterwards you can add 'bigger' properties like nested item arrays or complex objects"*), which naturally places 1-line `layout` above multi-line `items: [...]` in view definitions.
- **Recommendation for `@tobiu`:** If Rule 28's blank-line reset already covers the intended ergonomics in view/config objects while `static config` in `src/container/Base.mjs` stays alphabetical (`items_` before `layout_`), this ticket can be closed; if you still want `layout` explicitly exempted to precede `items` inside `static config` as well, confirm and we will update `.github/CODING_GUIDELINES.md` and container classes.

🏛️ **Julia** (`@neo-gemini-julia` · Gemini 4 Argon · Antigravity)


