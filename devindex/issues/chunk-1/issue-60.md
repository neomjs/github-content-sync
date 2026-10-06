---
id: 60
title: 'Deploy dist/development, dist/esm and dev mode beside dist/production'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-06T12:26:30Z'
updatedAt: '2026-10-06T12:40:39Z'
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
---
# Deploy dist/development, dist/esm and dev mode beside dist/production

## Context

The engine's Pages deploy serves every example and app in four environments: dev mode, `dist/development`, `dist/esm` and `dist/production`. On 2026-10-06 the operator asked whether DevIndex does the same since it moved to its own deploy (#27).

## The Problem

It deploys one. Measured on neomjs.com on 2026-10-06:

| URL under `neomjs.com/devindex/` | status |
|---|---|
| `` (root entry) and `deploy-receipt.json` | 200 |
| `dist/production/apps/devindex/index.html` | 200 |
| `dist/development/apps/devindex/index.html` | 404 |
| `dist/esm/apps/devindex/index.html` | 404 |
| `apps/devindex/index.html` (dev mode) | 404 |

The builds already exist. `pages.yml` runs `npm run build-all -- -l no -p no`, and with no `-e` the engine's `build-all` builds every environment. `buildScripts/assemblePagesSite.mjs:69` then copies only `dist/production` into the site, so `dist/development` and `dist/esm` are built and discarded on every deploy.

The Portal follows suit. All four engine registries (`apps/portal/resources/data/examples_{devmode,dist_dev,dist_esm,dist_prod}.json`, neomjs/neo#19184) link the DevIndex card to `https://neomjs.com/devindex/`. So the Portal's dev-mode, `dist/development` and `dist/esm` tabs open the production build.

## The Architectural Reality

- **Design authority:** #27's Fix 1 scoped the workflow to "the production site (`dist/production` plus the app's resources and its contributor data)". No record weighs the other three environments.
- **Operator intent on record (2026-02-24):** he tested DevIndex in all three dist environments, found it "works fine", and that it "pulls in data from dev mode. `Neo.config.basePath` is a lifesaver".
- **@neo-gpt's 2026-08-20 pages audit:** extracted apps become independent cells, with build artifacts kept per (neo revision, environment).
- `assembleSite()` writes one root entry, `<base href="./dist/production/">`, and one `neo-config.json` (`basePath: '../../'`, `workerBasePath: './'`), so both the page and the workers reach the mount. The contributor index (24 MB) sits once at `apps/devindex/resources/data/`, and every environment can share it through its `basePath`.
- Dev mode serves unbundled sources: the app's `apps/devindex/**` and the engine package files its entry and imports reach (`node_modules/neo.mjs/src/**` and the resources they load), under the mount.

## The Fix

`assembleSite()` emits all four environments under the mount:
- `dist/development` and `dist/esm` beside `dist/production`, each with its own `neo-config.json` whose `basePath` reaches the mount;
- dev mode as the app source plus the engine files it loads;
- the root entry stays production, and the contributor index stays one shared copy.

`deploy-receipt.json` lists the environments it shipped. The Portal registries then point each tab at its environment; that is an engine-side follow-up PR in neomjs/neo.

## Acceptance Criteria

- [ ] The four URLs in the table return 200 on neomjs.com after a deploy.
- [ ] Each environment boots in a browser and paints the contributor grid from the shared index.
- [ ] The assembly fails, instead of shipping, when any of the four builds is missing; a spec in `AssemblePagesSite.spec.mjs` pins it.
- [ ] `deploy-receipt.json` names the environments shipped.

## Out of Scope

- The data pipeline and the neomjs.com proxy routes.
- The Portal registry rows: a neomjs/neo PR after this one deploys.

## Related

#27 (the Pages deploy), neomjs/neo#19184 (the Portal's DevIndex rows), neomjs/neo#19108, D#19050 (devindex's own deploy).

Live latest-open sweep: neomjs/devindex has 2 open issues (#1, #9) at 2026-10-06T12:25Z; no equivalent.
A2A claim sweep: last 30 messages; no overlapping claim.
MC sweep: "devindex Pages deploy only dist/production …", 5 results. It surfaced the operator's 2026-02-24 all-environments test and the 2026-08-20 pages audit; neither is a decision against the other environments.
Own-assignment sweep (devindex): 1 open (#1, column resize); not overlapping.

Origin Session ID: 40a3c119-6419-4e87-9248-002c686708fe


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

