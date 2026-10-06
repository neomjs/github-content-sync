---
id: 60
title: 'Pages ships dist/development and dist/production, each from its own entry'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T12:26:30Z'
updatedAt: '2026-10-06T16:42:53Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/60'
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
closedAt: '2026-10-06T16:42:53Z'
---
# Pages ships dist/development and dist/production, each from its own entry

## Context

The engine's Pages deploy serves every example and app in four environments: dev mode, `dist/development`, `dist/esm` and `dist/production`. On 2026-10-06 the operator asked whether DevIndex does the same since it moved to its own deploy (#27), and agreed that all of them belong on the demo.

*Re-scoped 2026-10-06:* this ticket ships the two dist builds that boot under the mount. dist/esm and dev mode need fixes outside the assembly and moved to #61.

## The Problem

It deploys one. Measured on neomjs.com on 2026-10-06:

| URL under `neomjs.com/devindex/` | status |
|---|---|
| `` (root entry) and `deploy-receipt.json` | 200 |
| `dist/production/apps/devindex/index.html` | 200 |
| `dist/development/apps/devindex/index.html` | 404 |
| `dist/esm/apps/devindex/index.html` | 404 |
| `apps/devindex/index.html` (dev mode) | 404 |

The builds already exist. `pages.yml` runs `npm run build-all -- -l no -p no`, and with no `-e` the engine's `build-all` builds every environment. `buildScripts/assemblePagesSite.mjs` then copied only `dist/production` into the site, so `dist/development` was built and discarded on every deploy.

Copying a build is not enough under a mount, though. Measured in headless Chromium with the site served at `/tmp/devindex/`: a build's own entry (`dist/<env>/apps/devindex/index.html`) boots but paints no grid. Its workers run from `dist/<env>/`, two levels above the page, and resolve `basePath: '../../../../'` against their own script, so the index fetch climbs past the mount to the origin root and 404s. At an origin root the excess `../` is clamped, which is why the same build works locally. On neomjs.com the production entry from the table does paint, but only because the climb lands on `https://neomjs.com/apps/devindex/resources/data/users.jsonl`, the engine site's copy outside the mount (headless probe, 2026-10-06). That copy goes once the engine's Pages deploy runs on 13.2, which no longer ships `apps/devindex` (neomjs/neo#17240).

The Portal follows suit: all four engine registries (`apps/portal/resources/data/examples_{devmode,dist_dev,dist_esm,dist_prod}.json`, neomjs/neo#19184) link the DevIndex card to the site root.

## The Architectural Reality

- `assembleSite()` already solves this for the site root: its entry sets `<base href="./dist/production/">`, the directory the workers are served from, and the build's `neo-config.json` sits there with `basePath: '../../'` and `workerBasePath: './'`, so `basePath` names the mount from both sides.
- The contributor index (24 MB) and the guides sit once at the site root, and every entry reaches them through `basePath`.

## The Fix

`assembleSite()` takes the workspace root and ships each build in `ENTRIES` (`dist/development`, `dist/production`) whole. Each build's entry gets the root entry's treatment with `<base href="../../">`, its build directory, and the build's `neo-config.json` gets `basePath: '../../'`, `workerBasePath: './'` and `isGitHubPages`. The site root stays production's entry, and `deploy-receipt.json` lists the entries it shipped.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| The build entries `dist/development/apps/devindex/` and `dist/production/apps/devindex/` as served pages. Their consumers are the Portal's `examples_dist_dev.json` / `examples_dist_prod.json` rows, once the follow-up lands (#61) | this ticket; the root entry's existing treatment (`rootEntry` → `entryPage`) | Each page carries `<base href="../../">` (its build directory) and loads `src/MicroLoader.mjs`, which reads `dist/<env>/neo-config.json`. The index comes from `<mount>apps/devindex/resources/data/users.jsonl`, the guides from `<mount>learn/` | A build entry of another shape fails the assembly: `entryPage` throws on any rewrite that does not match. The site root stays production's entry | `assembleSite` / `entryPage` JSDoc | `AssemblePagesSite.spec.mjs`; a headless probe of the assembled site under a mount; the live read after the deploy (#61) |
| `dist/development/neo-config.json` (new) and `dist/production/neo-config.json` | the root entry's config contract | The build's app-entry config, with `basePath: '../../'`, `workerBasePath: './'` and `isGitHubPages: true` | Production's file serves both the site root and production's build entry, with the same content | JSDoc | spec: each build's config, and its `basePath` resolving to the mount from the build directory |
| `deploy-receipt.json` → `entries` (new field) | this ticket | `{"dist/development": "<publicBase>dist/development/apps/devindex/", "dist/production": "<publicBase>dist/production/apps/devindex/"}` beside the existing fields | Additive. No code reads the receipt today: `git grep deploy-receipt` across devindex, neo and pages finds only its writer and a `pages.yml` comment | JSDoc | spec: `receipt.entries` and the written file |
| `assembleSite({root, dataFile, out, publicBase, receipt})` and `entryPage(html, base)`, replacing `{build, learnDir, imagesDir, …}` and `rootEntry(html)` | this ticket | Takes the workspace root, ships each build in `ENTRIES` whole, and refuses a missing build before writing anything | Callers are the CLI block, which `pages.yml:102` invokes unchanged, and the spec. `pullDevIndexData.mjs` imports only `SITE_DATA`, which is unchanged | JSDoc | spec; `npm run test-unit` |

## Acceptance Criteria

- [ ] Under a mount, `dist/development/apps/devindex/` and `dist/production/apps/devindex/` paint the contributor grid and the learn view from the shared index and guides, like the site root (headless probe of the assembled site).
- [ ] The assembly fails, instead of shipping, when a build in `ENTRIES` is missing; `AssemblePagesSite.spec.mjs` pins it, together with each entry's base and config.
- [ ] `deploy-receipt.json` names the entries shipped.
- ~~After a deploy, the same on neomjs.com.~~ Moved on 2026-10-06 to #61, which owns every environment going live (Residual-Owner: #61). The neomjs/neo PR pointing the Portal's `dist_dev` and `dist_prod` rows at these entries moved there too.

## Out of Scope

- dist/esm and dev mode (#61, blocked by the bump to `neo.mjs` 13.2.0 and neomjs/neo#19430).
- The live read after the deploy, and the Portal registry rows (#61).
- The data pipeline and the neomjs.com proxy routes.

## Related

#27 (the Pages deploy), #61, neomjs/neo#19430, neomjs/neo#19184 (the Portal's DevIndex rows), neomjs/neo#19108, D#19050 (devindex's own deploy).

Live latest-open sweep: neomjs/devindex had 2 open issues (#1, #9) at 2026-10-06T12:25Z; no equivalent.
A2A claim sweep: last 30 messages; no overlapping claim.
MC sweep: "devindex Pages deploy only dist/production …", 5 results. It surfaced the operator's 2026-02-24 all-environments test and the 2026-08-20 pages audit; neither is a decision against the other environments.
Own-assignment sweep (devindex): 1 open (#1, column resize); not overlapping.


## Timeline

- 2026-10-06T12:26:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-06T12:26:31Z @neo-opus-grace added the `enhancement` label
- 2026-10-06T12:26:31Z @neo-opus-grace added the `ai` label
- 2026-10-06T12:27:10Z @neo-opus-grace cross-referenced by #17
### @neo-opus-grace - 2026-10-06T12:40:39Z

**Design authority, confirmed by the operator (2026-10-06):** all four environments. DevIndex is a product on its own, where `dist/production` would suffice, and it is also a demo app.

The Portal shows every example in four environments: `apps/portal/view/examples/TabContainer.mjs` has the tabs `dist/prod`, `dist/esm`, `dist/dev` and `Dev Mode`, each backed by its own registry (`examples_dist_prod.json`, `examples_dist_esm.json`, `examples_dist_dev.json`, `examples_devmode.json`). Once this deploys, the DevIndex row in each registry points at its own environment; that is the neomjs/neo follow-up named in Out of Scope.


- 2026-10-06T15:09:18Z @neo-opus-grace cross-referenced by #19430
- 2026-10-06T15:11:06Z @neo-opus-grace cross-referenced by #61
- 2026-10-06T15:11:33Z @neo-opus-grace changed title from **Deploy dist/development, dist/esm and dev mode beside dist/production** to **Pages ships dist/development and dist/production, each from its own entry**
- 2026-10-06T15:12:47Z @neo-opus-grace cross-referenced by PR #62
- 2026-10-06T16:42:53Z @tobiu referenced in commit `55ce8d0` - "Merge pull request #62 from neomjs/grace/60-all-envs

feat(pages): the site ships dist/development and dist/production, each from its own entry (#60)"
- 2026-10-06T16:42:54Z @tobiu closed this issue

