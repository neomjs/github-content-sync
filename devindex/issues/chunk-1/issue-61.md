---
id: 61
title: dist/esm and dev mode join the Pages site once they boot under the mount
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T15:11:05Z'
updatedAt: '2026-10-06T15:11:06Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/61'
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
# dist/esm and dev mode join the Pages site once they boot under the mount

## Context

Split from #60 on 2026-10-06. #60 ships dist/development and dist/production, each from its own entry. The other two environments the Portal links, dist/esm and dev mode, cannot boot under the `/devindex/` mount yet, for reasons outside the assembly.

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
  
  Its theme CSS comes from `dist/development/css`, which the site already ships.
- **The engine version.** The lock holds `neo.mjs` 13.1.0. Dependabot's grouped bump moves it to 13.2.0 about three days after the publish, and 13.2's `build-all` regenerates the engine's browser bundles before it builds.

## The Fix

1. After the bump to 13.2.0, dist/esm joins `ENTRIES` in `buildScripts/assemblePagesSite.mjs`, its entry kept as built with `isGitHubPages` set, plus FontAwesome at the site root.
2. After neomjs/neo#19430 ships, dev mode joins with the copy set above.

## Acceptance Criteria

- [ ] dist/esm's entry paints the grid from the shared index under the mount (headless probe) and on neomjs.com after a deploy.
- [ ] Dev mode's entry does the same, from the site's index rather than any copy inside `node_modules`.
- [ ] `deploy-receipt.json`'s `entries` names all four environments.
- [ ] A neomjs/neo PR points the Portal's dist/esm and dev-mode DevIndex rows at their entries.

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

