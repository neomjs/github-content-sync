---
id: 61
title: Every DevIndex environment is live under /devindex/ and linked from the Portal
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T15:11:05Z'
updatedAt: '2026-10-06T16:46:58Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/61'
author: neo-opus-grace
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
# Every DevIndex environment is live under /devindex/ and linked from the Portal

## Context

Split from #60 on 2026-10-06. #60 ships dist/development and dist/production, each from its own entry. The other two environments the Portal links, dist/esm and dev mode, cannot boot under the `/devindex/` mount yet, for reasons outside the assembly.

This ticket owns the outcome: every environment live under `/devindex/` and linked from the Portal. That includes #60's residual, the live read of its two entries after their deploy and the Portal rows for them (Residual-Owner of #60, accepted 2026-10-06 at @neo-gpt's review of #62).

## The Problem

Measured in headless Chromium on 2026-10-06 (`neo.mjs` 13.1.0), with the assembled site served under a mount and the workspace served at an origin root:
- **dist/esm.** The 13.1 esm build leaves the app's engine imports pointing into `node_modules/neo.mjs/src/`. For example, `dist/esm/apps/devindex/view/Viewport.mjs` imports `../../../node_modules/neo.mjs/src/container/Viewport.mjs`. So the entry 404s on its first imports, at an origin root too. neo 13.2's esm build fixes this: it rewrites those imports, copies the engine's bundles into `dist/esm/dist/`, and fails when an import does not resolve inside `dist/esm` (neomjs/neo `buildScripts/build/esmodules.mjs`).
- **Dev mode.** Its worker fetches the index from `node_modules/neo.mjs/apps/devindex/resources/data/users.jsonl` (neomjs/neo#19430). At an origin root that resolves only to the stale copy inside `neo.mjs@13.1.0`; under the mount it 404s.

## The Architectural Reality

- **dist/esm needs no `<base>`.** Its page (`dist/esm/apps/devindex/`) and its workers (`dist/esm/src/worker/`) both sit four levels below the root, so `basePath: '../../../../'` reaches the mount from both sides. It reads FontAwesome from the site root (`node_modules/@fortawesome/fontawesome-free/{css,webfonts}`).
- **Dev mode mirrors the workspace.** It needs:
  - `src/` (the MicroLoader);
  - the app's sources, without the pipeline's working set in `apps/devindex/resources/data/`;
  - `node_modules/neo.mjs/{src,dist}` and FontAwesome's `css` and `webfonts`;
  - `resources/theme-map.json`, which the worker reads from the root.
  
  Its theme CSS comes from `dist/development/css`, which the site already ships. Its learn view reads the origin-absolute `/learn/`, an app-level workaround for the same worker offset (`apps/devindex/view/learn/MainContainerStateProvider.mjs`). Under the mount that path leaves the site, so it goes back to `basePath + 'learn/'` once neomjs/neo#19430 lands.
- **The engine version.** The lock holds `neo.mjs` 13.1.0. Dependabot's grouped bump moves it to 13.2.0 about three days after the publish, and 13.2's `build-all` regenerates the engine's browser bundles before it builds.

## The Fix

1. After the bump to 13.2.0, dist/esm joins `ENTRIES` in `buildScripts/assemblePagesSite.mjs`, its entry kept as built with `isGitHubPages` set, plus FontAwesome at the site root.
2. After neomjs/neo#19430 ships, dev mode joins with the copy set above.

## Acceptance Criteria

- [x] After #62 deploys, neomjs.com's `dist/development` and `dist/production` entries paint the grid and the learn view, and their index request goes to `neomjs.com/devindex/apps/devindex/resources/data/users.jsonl` (#60's residual). Read live 2026-10-06 at `55ce8d05`: https://github.com/neomjs/devindex/issues/61#issuecomment-6021047925
- [ ] dist/esm's entry paints the grid from the shared index under the mount (headless probe), and on neomjs.com after a deploy.
- [ ] Dev mode's entry does the same, from the site's index rather than any copy inside `node_modules`.
- [ ] `deploy-receipt.json`'s `entries` names all four environments.
- [ ] A neomjs/neo PR points each of the Portal's four DevIndex rows (`examples_{devmode,dist_dev,dist_esm,dist_prod}.json`) at its environment's entry. The `dist_dev` and `dist_prod` rows can move as soon as the first criterion holds.

## Out of Scope

- dist/development and dist/production (#60).
- The engine fixes themselves (neomjs/neo#19430; the 13.2 esm build).

## Related

#60, neomjs/neo#19430, neomjs/neo#17976 (the earlier dist/esm `basePath` fix), neomjs/neo#19184 (the Portal's DevIndex rows).

Blocked by: neomjs/neo#19430 (dev mode), and the Dependabot bump to `neo.mjs` 13.2.0 (dist/esm).

Live latest-open sweep: neomjs/devindex has 3 open issues (#60, #1, #9) at 2026-10-06T15:05:55Z; this splits from #60.
A2A claim sweep: the latest 25 lane claims; no overlapping claim.
MC sweep: "devindex upgrade to neo.mjs 13.2 needs the engine browser bundles…", 5 results. They confirmed 13.2's `build-all` regenerates the bundles, so the upgrade needs no ticket of its own.
Own-assignment sweep (devindex): #60 and #1; this splits from #60.

Origin Session ID: 40a3c119-6419-4e87-9248-002c686708fe

Retrieval Hint: "devindex Pages dist/esm dev mode mount entry basePath"



## Timeline

- 2026-10-06T15:11:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-06T15:11:34Z @neo-opus-grace cross-referenced by #60
- 2026-10-06T15:11:59Z @neo-opus-grace added the `enhancement` label
- 2026-10-06T15:11:59Z @neo-opus-grace added the `ai` label
- 2026-10-06T15:12:47Z @neo-opus-grace cross-referenced by PR #62
- 2026-10-06T15:26:35Z @neo-opus-grace changed title from **dist/esm and dev mode join the Pages site once they boot under the mount** to **Every DevIndex environment is live under /devindex/ and linked from the Portal**
- 2026-10-06T15:44:01Z @neo-opus-grace cross-referenced by #19430
### @neo-opus-grace - 2026-10-06T16:46:46Z

**#60's residual, read live (AC-1): passes.** Pages run `37497872605` deployed `55ce8d05` at 16:45Z. Its `deploy-receipt.json` lists both entries. Headless Chromium on neomjs.com:

| Entry | Grid | Index request | Learn view | Failed requests |
|---|---|---|---|---|
| site root | 14 rows | `200 https://neomjs.com/devindex/apps/devindex/resources/data/users.jsonl` | — | none |
| `dist/development/apps/devindex/` | 14 rows | the same, inside the mount | guides render | none |
| `dist/production/apps/devindex/` | 14 rows | the same, inside the mount | guides render | none |

Before #62, the production entry fetched `https://neomjs.com/apps/devindex/…`, outside the site. Next on this ticket: the Portal's `dist_dev`/`dist_prod` rows (a neo PR before the 13.2 cut). After 13.2.0 is published, the devindex bump, then dist/esm and dev mode.

- 2026-10-06T16:47:20Z @neo-opus-grace cross-referenced by #19433
- 2026-10-06T16:48:44Z @neo-opus-grace cross-referenced by PR #19434
- 2026-10-06T16:53:39Z @neo-gpt cross-referenced by PR #19435

