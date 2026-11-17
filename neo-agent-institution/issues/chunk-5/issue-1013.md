---
id: 1013
title: The Institution's theme sheets become plain CSS with the engine's 14.0
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-11-17T04:34:46Z'
updatedAt: '2026-11-17T04:34:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1013'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 21217 Theme sources become plain CSS, and the theme build copies them'
blocking: []
---
# The Institution's theme sheets become plain CSS with the engine's 14.0

## Context

The engine's theme sources become plain CSS in 14.0 (neomjs/neo#21217, graduated from neomjs/neo Discussion #18182), and Sass leaves the engine. This repository follows `github:neomjs/neo#dev` and builds its own themes with the engine's `buildScripts/build/themes.mjs`, so its sheets stop building the moment that change merges. This ticket converts them, ready to merge the same day (neomjs/neo#21217, AC-7).

Measured with the #18182 converter at this repository's `dev` `a4d1ed23c6`: all 59 non-partial files under `resources/scss` convert to plain CSS whose Sass output equals what the original compiles to, normalized and through autoprefixer (neomjs/neo Discussion #18182, comment 19670385). Six top-level rules take Sass's flattened output in place there: three because native nesting would raise a specificity, and three for a declaration after a nested node.

## The Problem

`package.json` runs `node ./node_modules/neo.mjs/buildScripts/build/themes.mjs` for `build-themes` and the engine's `watchThemes.mjs` for `watch-themes`. After neomjs/neo#21217 they read `resources/css` and copy `.css` files. This repository's 61 files under `resources/scss` (59 compiled, 2 partials) would not be built at all.

## The Architectural Reality

- `resources/scss/**`: 61 files. Two use the mixin `scroll-fade` with an argument (`fleet/instances/MemoryCandidateList.scss`, `setup/CreateContainer.scss`); the converter inlines it with its argument bound.
- `package.json` scripts `build-themes` and `watch-themes` call the engine's scripts; `server-start` and `server-start-ci` belong to neomjs/neo#21215 and are not touched here.

## The Fix

Run the converter from neomjs/neo#21217's pull request head over this repository's `dev`, commit `resources/css/**` in place of `resources/scss/**`, and check the result the way the engine leaf does.

## Acceptance Criteria

- [ ] AC-1: every converted file compiles to what its original compiles to (empty normalized diff), the converter's negative control passing; the pull request carries the run.
- [ ] AC-2: with neomjs/neo#21217's head installed, `npm run build-themes` builds `dist/development/css` from `resources/css`, and the Fleet Manager renders styled, checked in a browser.
- [ ] AC-3: merged the same day as neomjs/neo#21217, and not before it.

## Out of Scope

- The `server-start` scripts (neomjs/neo#21215).
- Re-nesting the six flattened rules by hand.

## Related

neomjs/neo#21217 · neomjs/neo Discussion #18182 · neomjs/neo#21215

Live latest-open + closed sweep: 59 open (red band); open and closed searches ("scss css theme sass") at 2026-11-17T04:37Z found nothing on this tree. A2A sweep: no claim. Ownership: self-assigned, as the converter's owner and steward of the paired merge.

Origin Session ID: e9f42310-21e7-4c15-aada-079042a61067
Retrieval Hint: "Institution theme sheets plain CSS paired with the engine's 14.0 Sass removal"


## Timeline

- 2026-11-17T04:34:47Z @neo-opus-vega added the `enhancement` label
- 2026-11-17T04:34:47Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-17T04:34:47Z @neo-opus-vega added the `ai` label
- 2026-11-17T04:34:47Z @neo-opus-vega added the `build` label
- 2026-11-17T04:34:57Z @neo-opus-vega marked this issue as being blocked by #21217
- 2026-11-17T04:40:39Z @neo-opus-vega cross-referenced by #21217
- 2026-11-17T05:56:20Z @neo-opus-vega cross-referenced by PR #21225
- 2026-11-17T06:01:09Z @neo-opus-vega cross-referenced by PR #1014
- 2026-11-17T11:19:53Z @neo-opus-vega referenced in commit `34d457c` - "refactor(themes): the family-ink mapping lives once, and the scroll-fade twins are pinned (#1013)

The conversion left two Sass sharings as copies. The family-ink mapping
moves into FamilyRailComponent.css as flat lists naming the rail and the
agent card monogram; the card always mounts a FamilyRail, so the sheet is
loaded beside it. The scroll-fade consumers share no loaded sheet, so each
keeps the block between markers, in one flat form, and
scrollFadeParity.spec.mjs fails when the two diverge. Flattened, both
scroll-fade sheets keep their rule sequence and the family-ink pair its
rules."
- 2026-11-17T11:19:53Z @neo-opus-vega referenced in commit `06b6a94` - "docs(themes): the comments and docs name the CSS tree (#1013)

Fifty files still named .scss sheets, resources/scss or "the SCSS" after the
conversion. File names and paths follow the tree; the engine motion file is
the engine's src/Global.css now; CONTRIBUTING says build-themes copies the
sheets. The vendored highlight changelog, design HTML and the lockfile keep
their wording."
- 2026-11-17T11:19:54Z @neo-opus-vega referenced in commit `38d1fa6` - "test(visual): the baseline stamp names the converted inputs (#1013)

test-goldens against the engine at neomjs/neo#21225 (4980924e0e): visual 63
of 63, e2e 2 of 2, e2e:nl 13 of 14. The one red arm (FleetGoldenPathNL, pane
width 909 for 988) fails the same way on Institution dev with the engine dev,
so it is not this change."
- 2026-11-17T11:27:12Z @neo-opus-vega referenced in commit `2614990` - "test(e2e): the rail drawer spec keeps its reason and drops a tracking ref (#1013)

The comment sweep touched this spec, and the archaeology guard reads every
touched file whole, so the comment's old "#73 item 5" ref failed the PR
baseline. The reason for the settle-aware wait stays."

