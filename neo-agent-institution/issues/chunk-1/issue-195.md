---
id: 195
title: 'Engine pin 11: dev@87ac80a6 carries the prepare dependency-build guard'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T10:54:22Z'
updatedAt: '2026-09-25T11:19:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/195'
author: neo-fable-clio
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T11:19:51Z'
---
# Engine pin 11: dev@87ac80a6 carries the prepare dependency-build guard

## Context

`npm run dist` (the Electron packaging under #7) dies in `harness/pack.mjs`'s stage install on the current pin (`neo.mjs: github:neomjs/neo#d85060795d…`): npm builds the git-pinned engine in a cache clone and runs its `prepare` there with `INIT_CWD` = the stage root, so husky and the skills materializer wrote into the stage and raced the Brain's `postinstall` (`EEXIST symlink … .stage/organism/.claude/skills/architecture-pre-flight`). The engine fixed the cause today: neomjs/neo#19204 → PR neomjs/neo#19205, merged 2026-09-25 10:51Z as `dev@87ac80a68edde7597d017436796440821c80cf7b` — `runPrepare` skips both stages when `INIT_CWD` is not the engine's own checkout. The Institution reaches that guard only through its pin.

## The Problem

The pack stage installs the engine from the product's pin (`buildOrganismManifest`: the Engine is the product's pin), so the fix is inert for this repository until the pin names a commit at or after `87ac80a6`. Between `d850607` and `87ac80a6` lie 128 engine commits (53 under `src/`, 13 under `buildScripts/`; no file removed or renamed under `src/`), among them the canvas worker per window group (neomjs/neo#19125), engine renderers loading from the package in a workspace build (neomjs/neo#19165), the canvas context hook and pointer bridge (neomjs/neo#19169, #19179), dock and grid fixes — an engine pin bump re-renders every golden and can move a consumer seam, so it is its own leaf with its own receipts (pins 7, 9 and 10 each surfaced one).

**Second writer, found on the pinned tree (2026-09-25 13:00 local):** with the pin at `87ac80a6` the stage install still dies, at `node_modules/neo-agent-brain/node_modules/neo.mjs` → `sh -c neo-agent-skills-materialize`. The Brain declares its own engine as a tarball, `https://github.com/neomjs/neo/archive/17b59aad8f….tar.gz` (2026-08-28), whose `package.json` still carries `postinstall: neo-agent-skills-materialize` (the engine moved that call to `prepare` on 2026-09-23, neomjs/neo#19053). The two specs differ, so npm nests a second engine under the Brain, and its postinstall materializes into `INIT_CWD` — the stage root — racing the Brain's own `postinstall` (EEXIST on `.agents/skills`). A tarball install runs no `prepare`, so the engine's guard never sees it.

## The Architectural Reality

- `package.json` + `package-lock.json` hold the pin; `node_modules/neo.mjs` is a github-SHA install with no `.git` and no `gitHead` — the pin is verified by a code marker (`grep -c dependency-build node_modules/neo.mjs/buildScripts/util/prepare.mjs` → 2), never by version.
- `buildOrganismManifest` already names the Engine as the product's pin (`OWNER_EXCEPTIONS`) and repeats both owners' `overrides` into the lockless stage manifest — but it never forced the Brain's `neo.mjs` edge onto that pin, so the stage carried two engines whenever the specs differed.
- The Isolated Institution CI job's `check-visual-baselines` is an input-identity stamp over `apps/agentos`, `resources/scss`, the capture specs and the engine lock entry: a pin bump moves the lock, so the stamp is re-issued after staging, and the goldens are re-rendered by `test-visual` because the stamp cannot see pixels.
- The Brain checkout's `node_modules/neo.mjs` is a symlink into this repository's install; replacing the install in place keeps it valid.

## The Fix

1. Pin → `github:neomjs/neo#87ac80a68edde7597d017436796440821c80cf7b`, lock updated by `rm -rf node_modules/neo.mjs && npm install`, marker verified; the drift check (removed engine members ∩ used identifiers, the unit tree with the Brain root, components, `test-visual` alone with any moved golden read by eye and updated in the same commit); stamp re-issued after staging.
2. `buildOrganismManifest` forces one engine: `overrides['neo.mjs']` = the product's pin, so the Brain's own engine spec resolves to the same install and no nested engine lifecycle runs in the stage; a declared override naming another engine fails the pack (`pack.spec.mjs` pins both arms).
3. `npm run dist` completes with the stage installed from the pin and no lifecycle workaround (the receipt #7 has carried as a residual since PR #150).

The Brain's own side — its `postinstall` materializer and the 08-28 engine tarball pin — is filed in neomjs/neo-agent-brain for its owners; this leaf makes the pack correct regardless.

## Acceptance Criteria

- [ ] `package.json` / `package-lock.json` pin the engine at `87ac80a68edde7597d017436796440821c80cf7b` (or a later `dev` commit) and the marker reads 2.
- [ ] `test/playwright/unit/` green with `NEO_AGENTOS_RUNTIME_ROOT` set; components green.
- [ ] `npm run test-visual` alone: every drifted golden updated in this leaf with a by-eye reason (content vs chrome), none left stale.
- [ ] `npm run dist` on this head stages the organism from the pin and emits `dist-artifacts/mac-arm64/Neo Harness.app` without `npm_config_ignore_scripts` (receipt: the pack log's `organism staged … rebuilt=true` + the artifact path); the staged `node_modules` holds exactly one `neo.mjs`.
- [ ] The baseline stamp is re-issued after the pin, lock and goldens are staged; `check-visual-baselines` exits 0 on the pushed head.

## Out of Scope

- The packaged app's plane-attach configuration without env (#12's first-run surface) and the team-instance checkout (Vega's leaf under #7).
- The Brain's `postinstall` → `prepare` move and its engine pin bump (the Brain ticket).
- Pinning this repository's engine as a tarball instead of a git commit (Ada's note on neomjs/neo#19205: skips clone, devDependencies and `prepare` altogether) — its own leaf.

## Related

#7 (parent) · neomjs/neo#19204 · neomjs/neo#19205 · neomjs/neo#19053 · #191 / #192 · #193 / #194 · pins 9 (#136) and 10 (#151)

Live latest-open sweep: all 17 open issues checked at 2026-09-25 10:53Z; no equivalent. A2A claim sweep: no claim on the pin. Memory Core: pins 7/9/10 recipes recalled.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "Institution engine pin 11 87ac80a6 prepare dependency-build guard pack stage one engine override"


## Timeline

- 2026-09-25T10:54:23Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T10:54:24Z @neo-fable-clio added the `enhancement` label
- 2026-09-25T10:54:24Z @neo-fable-clio added the `ai` label
- 2026-09-25T10:54:25Z @neo-fable-clio added the `build` label
- 2026-09-25T10:54:54Z @neo-fable-clio added parent issue #7
- 2026-09-25T11:05:10Z @neo-fable-clio referenced in commit `c153d7d` - "fix(harness): the stage manifest forces one engine, the product's pin (#195)

With the pin at 87ac80a6 the stage install still died, at
node_modules/neo-agent-brain/node_modules/neo.mjs → neo-agent-skills-materialize:
the Brain pins its own engine as a tarball (17b59aad8f, 2026-08-28) whose package.json
still materializes from postinstall, npm nests that second engine under the Brain
because the specs differ, and its postinstall writes into INIT_CWD, the stage root,
racing the Brain's own postinstall on .agents/skills. The Engine is the product's pin,
so buildOrganismManifest now forces every neo.mjs edge onto it through overrides; a
declared override naming another engine fails the pack."
- 2026-09-25T11:05:42Z @neo-fable-clio cross-referenced by #482
- 2026-09-25T11:08:09Z @neo-fable-clio cross-referenced by PR #196
- 2026-09-25T11:19:51Z @tobiu referenced in commit `cf56a1e` - "Merge pull request #196 from neomjs/fix/195-engine-pin-11

build(deps): engine pin 11 → dev@87ac80a6, and the stage manifest forces one engine (#195)"
- 2026-09-25T11:19:51Z @tobiu closed this issue
- 2026-09-25T11:54:58Z @neo-opus-grace cross-referenced by PR #484

