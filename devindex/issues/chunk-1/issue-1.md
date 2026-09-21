---
id: 1
title: Column resize moves the header but never the body cells
state: OPEN
labels:
  - bug
  - ai
  - grid
assignees:
  - neo-opus-grace
createdAt: '2026-08-20T12:06:12Z'
updatedAt: '2026-08-20T12:06:28Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/1'
author: neo-opus-grace
commentsCount: 0
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
# Column resize moves the header but never the body cells

## Context

Operator report: column drag and resize break in this app. Reproduced locally and traced to a single cause — this repository consumes published `neo.mjs@13.1.0`, and the two fixes for exactly this defect landed on neo's `dev` **after** that release was cut.

**No code in this repository is at fault.** The app's own view layer is byte-identical to `neomjs/neo`'s copy apart from import paths (`GridContainer.mjs` 421/421 lines, `MainContainer.mjs` 63/63, `Heuristics.mjs` 97/97, `Viewport.mjs` 139/139, zero substantive diff lines). Filing it here because the app that misbehaves is this one, and the fix has to be *received* here.

Live latest-open sweep: 0 open issues in this repository at 2026-08-20T12:05:15Z; no equivalent possible. A2A in-flight claim sweep over the latest 30 messages: no overlapping `[lane-claim]`.

## The Problem

Measured at `ef0390a`, `neo.mjs@13.1.0` installed, Chrome at 1920×1080, driving `/apps/devindex/index.html` from this repo's own dev server. One column resized by dragging its right-edge handle 150px:

| arm | header width | first-row cell, **mid-drag** | first-row cell, **on drop** |
|---|---|---|---|
| **published `13.1.0`** (as shipped) | 60 → 210 | 60 → **60** | 60 → **60** |
| the same app against neo `dev` `src/` | 60 → 210 | 60 → **210** | 60 → **210** |

Only the engine differs between the two arms — same app code, same gesture, same harness, same browser. The gesture always lands (the header always resizes); the body cells never follow on `13.1.0`, neither live nor on drop. That is the operator's report exactly.

Symbol probe against the **installed package**, not inferred from git:

| symbol | origin | `node_modules/neo.mjs@13.1.0` | neo `dev` `src/` |
|---|---|---|---|
| `refreshColumns` | neomjs/neo#17289 — the live-repaint half | **0 files** | 2 files |
| `isMeasuredWidth` | neomjs/neo#17327 — width comes from config unless layout owns it | **0 files** | 1 file |

`refreshColumns` is the mechanism: without it a pure width change leaves the mounted column range unchanged, the config never notifies, and the cells keep the geometry that was just replaced.

**Timeline.** `v13.1.0` was cut 2026-07-03. neomjs/neo#17289 landed `d41f68520e` on 2026-08-17; neomjs/neo#17327 landed `3085d62855` on 2026-08-18. Both are ancestors of neo's `dev` and of neither `main`, so no published release carries them.

## The Architectural Reality

- `package.json` pins `"neo.mjs": "^13.1.0"` — a caret range, so a `13.2.0` publish satisfies it with no manifest edit
- `apps/devindex/view/home/GridContainer.mjs` — ~37 columns, component-backed cells; the shape is unusual but not implicated
- The engine surfaces are `grid/header/Toolbar.mjs` (`isMeasuredWidth`, `passSizeToBody`) and `grid/Body.mjs` (`refreshColumns`), both inside `node_modules/neo.mjs`

## The Fix

**Nothing changes in this repository's source.** Receive a `neo.mjs` release that carries both fixes:

1. `v13.2.0` (or later) is published from neo's `dev`
2. A fresh install / lockfile refresh here picks it up under the existing `^13.1.0` range
3. Re-run the reproduction above and confirm the cells track in both arms

neomjs/neo#17401 — component cells keeping their first record's content across a scroll — merged to neo `dev` today and affects this same grid. It should ride the same release rather than a second one.

## Acceptance Criteria

- [ ] The installed `neo.mjs` reports a version whose `src/` contains both `refreshColumns` and `isMeasuredWidth`
- [ ] Dragging a column's resize handle updates the body cell widths **during** the drag, not only on drop
- [ ] On drop, the following columns' positions settle against the new width
- [ ] The reproduction table above is re-run at the upgraded version and the `13.1.0` row is the only one showing stale cells
- [ ] Scrolling after the upgrade shows component-backed cells (`User`, `Impact`, `Top Repo`, `Location`) following their record — the neomjs/neo#17401 half

## Out of Scope

- Removing `apps/devindex` from `neomjs/neo` — tracked on neomjs/neo#17238, and gated on this ticket closing
- neomjs/devindex#9 (frozen opt-in cursor) — same app, unrelated mechanism
- Any change to this repo's grid or view code; there is no defect there to fix

## Avoided Traps

**Two measurement traps cost real time here; inherit them rather than re-paying.**

1. **A fresh clone renders zero grid rows, and it is not a bug.** `npm install` does not build the themes. Without `dist/development/css/**` the stylesheets 404, the grid body computes to `height: 0`, and row virtualization renders no rows — while the footer still reports `Visible Rows: 50,000`, which reads exactly like a data-layer fault. Build first:
   `node ./node_modules/neo.mjs/buildScripts/build/themes.mjs -f -n -e dev -t all`
   (`npm run build-themes` prompts interactively and cannot be scripted as-is.)
2. **The resize handle is injected on hover.** A drag started without first hovering the header button's right edge arms nothing and silently does a column *reorder* instead. Hover, confirm a `.neo-resizable` exists, then press.

## Related

- neomjs/neo#17289 — header→cell width sync on resize (the live-repaint half)
- neomjs/neo#17327 — a column's width comes from its config unless layout owns it
- neomjs/neo#17401 — component cells must follow their record across a scroll
- neomjs/neo#17409 — the original investigation, closed as not-a-defect **in that repository**; this ticket is its real home
- neomjs/neo#17238 — the epic whose terminal step is blocked by this

Origin Session ID: 3e4f33e0-fb23-4a61-a2a0-7f396950f3d6

Handoff Retrieval Hints: `query_raw_memories("devindex column resize published 13.1.0 refreshColumns")`. Commit anchors: neo `d41f68520e`, `3085d62855`; this repo at `ef0390a`.


## Timeline

- 2026-08-20T12:06:14Z @neo-opus-grace added the `bug` label
- 2026-08-20T12:06:14Z @neo-opus-grace added the `ai` label
- 2026-08-20T12:06:14Z @neo-opus-grace added the `grid` label
- 2026-08-20T12:06:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-20T12:07:09Z @neo-opus-grace cross-referenced by #2
- 2026-08-20T12:22:13Z @neo-opus-grace cross-referenced by #17409
- 2026-08-20T14:11:12Z @neo-gpt cross-referenced by PR #17417

