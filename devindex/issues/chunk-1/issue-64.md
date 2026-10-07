---
id: 64
title: 'DevIndex cannot build on neo.mjs 13.2: the Data Factory sits under apps/'
state: CLOSED
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-opus-grace
createdAt: '2026-10-07T10:56:26Z'
updatedAt: '2026-10-07T11:37:50Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/64'
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
blocking:
  - '[ ] 61 Every DevIndex environment is live under /devindex/ and linked from the Portal'
closedAt: '2026-10-07T11:37:50Z'
---
# DevIndex cannot build on neo.mjs 13.2: the Data Factory sits under apps/

## Context

A pre-cut witness for #61 installed Engine `dev` at neomjs/neo@b2db92c5b7 (packed the way the 13.2 publish packs it) into a DevIndex worktree at `bf6fc43` and ran the Pages workflow's own build, `npm run build-all -- -l no -p no`. It exits 1.

The cause is a deliberate engine change: neomjs/neo#17429 (merged 2026-08-20, after the 13.1.0 tag) removed the two DevIndex special cases from the engine when DevIndex left the neo repo. 13.2.0 is the first release without them. DevIndex still relies on both.

## The Problem

Measured on 2026-10-07:

1. **The build fails.** The app worker's webpack build reports 24 errors such as `Can't resolve 'tty'` in `node_modules/cli-width`, plus `string_decoder`, `stream`, `node:events`, `node:child_process` and `node:path`. Every error is reached through `./apps/ lazy ^\.\/.*\.mjs$` → `./devindex/services/{Manager,cli,GitHub,Storage,config}.mjs`. The engine's worker dynamic imports (`src/worker/{App,Canvas,Data}.mjs`) let webpack bundle every `.mjs` under `apps/` that their `webpackExclude` does not name. 13.1.0's regex named `devindex(?:\/|\\)services` (added in neomjs/neo#9583); 13.2's does not.
2. **Every build carries the contributor index.** 13.1's app-worker config deleted `apps/devindex/resources/data/` from the build output ("Exception for devindex app: Do not deploy the data folder"); 13.2 copies the app's `resources/` whole. After the build, `dist/development`, `dist/production` and `dist/esm` each hold the 23 MB `users.jsonl` that `postinstall` pulls. The site serves the index once, at `apps/devindex/resources/data/users.jsonl` under the mount. The copies are dead weight in every Pages artifact: two today, three once dist/esm ships.

DevIndex itself is unaffected while its lock holds `neo.mjs` 13.1.0. The first deploy after the bump to 13.2.0 fails.

## The Architectural Reality

- `apps/` is the browser tree. The engine's worker contexts scan it so an app's modules can load by path. Node-side code that sits there needs an engine exclusion, and a consumer-named exclusion is what #17429 rightly removed.
- `apps/devindex/services/` is the Data Factory (learn/data-factory): Node-only (`inquirer`, `child_process`, the GitHub API), run through the `devindex:*` npm scripts. No browser module imports it; its importers are `package.json`, `buildScripts/{assemblePagesSite,publishWorkingSet,pullDevIndexData}.mjs`, eight specs under `test/playwright/unit/app/devindex/` and the data-factory guides. Its own engine imports climb three levels (`../../../node_modules/neo.mjs/src/core/Base.mjs`).
- Precedent: neomjs/neo#12270 moved Node-only code out of a tree the worker contexts scan (`examples/cloud-deployment` → `ai/examples/`) rather than excluding it.
- Witness: with `apps/devindex/services/` moved outside `apps/`, the same build exits 0 (16.8 s, 0 webpack errors), and dist/esm's app imports contain no `node_modules/neo.mjs` path (23 importing modules, 0 hits).

## The Fix

1. Move `apps/devindex/services/` to `services/` at the workspace root, beside `buildScripts/`. Its engine imports become `../node_modules/neo.mjs/...`. Class names (`DevIndex.services.*`) stay. Update every importer: the six `devindex:*` scripts, the three build scripts, the eight specs and the guides' paths.
2. `assemblePagesSite.mjs` copies each build without `apps/devindex/resources/data/`, so the index ships once.

Both changes are inert on 13.1.0: the move leaves nothing for its exclusion to match, and the builds have no data folder to skip. So this lands before the publish, and the bump to 13.2.0 is a lock change only.

## Acceptance Criteria

- [ ] No `.mjs` under `apps/` imports a Node builtin or a Node-only package; the Data Factory lives in `services/`, and every `devindex:*` script, build script, spec and guide path follows it.
- [ ] `npm run build-all -- -l no -p no` exits 0 on the locked 13.1.0 and on a packed Engine `dev` tarball (the 13.2 candidate).
- [ ] The assembled `_site` holds `users.jsonl` once, at `apps/devindex/resources/data/`, and in no build directory (assembly spec).
- [ ] The unit suite passes.

## Out of Scope

- The dist/esm and dev-mode entries and the Portal rows (#61).
- The bump to 13.2.0 itself (Dependabot or the post-publish bump).
- Any engine change: the engine stays free of consumer names.

## Avoided Traps

- **Re-adding a DevIndex clause to the engine's `webpackExclude`.** It reverses #17429 and keeps a consumer's layout in four engine files.
- **`resolve.fallback: false` for Node builtins in the engine's webpack configs.** It would compile the Node-only modules into the worker chunks and hide every future browser import of a builtin until runtime.

## Decision Record impact

none

## Related

#61 (blocked by this), neomjs/neo#17429 (removed the exceptions), neomjs/neo#9583 (added the exclusion), neomjs/neo#12270 (the move precedent).

Live latest-open sweep: neomjs/devindex open #61, #1, #9 and neomjs/neo's latest 20 open issues at 2026-10-07T10:53Z, re-read 10:56Z; no equivalent. `gh search issues` for "devindex services webpack" and "13.2 devindex": none.
A2A claim sweep: the latest 30 messages; no claim on DevIndex's build or the 13.2 bump.
MC sweep: "devindex build-all fails webpack node builtins apps/devindex/services", 6 results. They show why the exclusion existed (neomjs/neo#9583) and the #12270 move precedent; no prior decision against moving the folder.
Own-assignment sweep (devindex): #61 and #1. #61 owns the entries, not the build.

Origin Session ID: 9aa8aa9b-2502-458b-976b-eec8a223218e

Retrieval Hint: "devindex neo.mjs 13.2 build-all webpack apps/devindex/services Data Factory move"


## Timeline

- 2026-10-07T10:56:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-07T10:56:28Z @neo-opus-grace added the `bug` label
- 2026-10-07T10:56:28Z @neo-opus-grace added the `ai` label
- 2026-10-07T10:56:28Z @neo-opus-grace added the `architecture` label
- 2026-10-07T10:56:35Z @neo-opus-grace marked this issue as blocking #61
- 2026-10-07T11:04:19Z @neo-opus-grace referenced in commit `c5eb555` - "test(devindex): the run-boundary preload comment drops its ticket ref (#64)"
- 2026-10-07T11:04:32Z @neo-opus-grace cross-referenced by PR #65
- 2026-10-07T11:06:26Z @neo-opus-grace cross-referenced by #14800
- 2026-10-07T11:08:46Z @neo-opus-grace cross-referenced by #61
- 2026-10-07T11:37:50Z @tobiu referenced in commit `db93e68` - "Merge pull request #65 from neomjs/grace/64-data-factory-out-of-apps

fix(build): the Data Factory leaves apps/, so DevIndex builds on neo.mjs 13.2 (#64)"
- 2026-10-07T11:37:50Z @tobiu closed this issue

