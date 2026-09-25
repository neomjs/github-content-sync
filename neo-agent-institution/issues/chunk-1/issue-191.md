---
id: 191
title: 'The Brain resolver imports ./src/Neo.mjs, absent from every Brain root'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - build
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T10:17:59Z'
updatedAt: '2026-09-25T10:53:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/191'
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
closedAt: '2026-09-25T10:53:51Z'
---
# The Brain resolver imports ./src/Neo.mjs, absent from every Brain root

## Context

Operator goal for 2026-09-25: the Fleet Manager's Electron shell installed on the team machine, every peer entering the same instance. `npm run start:brain` (and a packaged double-click) must first resolve the Brain's paths and its plane declaration before it can choose PLANE-ATTACH / ATTACH / OWN. On this machine that step fails before any `HARNESS_BRAIN_PLAN` line; reproduced 2026-09-25 by running the resolver's own script in the Brain checkout (`dev@6057492`):

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/Users/Shared/…/neo-agent-brain/src/Neo.mjs'
```

## The Problem

`harness/brain.mjs#resolveBrainPaths` (`brain.mjs:236-283`) spawns `node --input-type=module -e` with `cwd: repoRoot` (the Brain root) and this bootstrap:

```js
import Neo from './src/Neo.mjs';
import * as core from './src/core/_export.mjs';
import InstanceManager from './src/manager/Instance.mjs';
import AiConfig from './ai/config.mjs';
```

That is the pre-split shape (initial files, 2026-08-26), when `ai/` lived inside the engine repository and `./src/Neo.mjs` was the engine. Since the split the Brain root's `src/` holds `composition/ evolution/ fleet/` only; the Brain's own entrypoints bootstrap the engine from the package — `ai/daemons/orchestrator/daemon.mjs:17-27`: `import 'dotenv/config'`, `import Neo from 'neo.mjs/src/Neo.mjs'`, `import * as core from 'neo.mjs/src/core/_export.mjs'`, `import InstanceManager from 'neo.mjs/src/manager/Instance.mjs'`. The launcher was re-pointed at the Brain root (#4, #115, #43) without updating the resolver's imports, so `start:brain` and the packaged Brain boot fail on every post-split Brain root — checkout and staged organism alike (the stage's `src/` is the product's `src/MicroLoader.mjs` + the Brain's three trees; no `Neo.mjs`).

`test/playwright/unit/harness/brain.spec.mjs` does not cover the resolver script's shape (no `resolveBrainPaths` arm), which is how the drift stayed green.

## The Architectural Reality

- `resolveBrainPaths({repoRoot, env, execFileFn})` is the ONLY reader of the plane declaration (`AiConfig.fleet.planeBase`) and the mutable paths the harness supervises; `bootProductBrain` (`main.mjs:955-1027`) and the smoke both consume it. It reads AiConfig leaves through the Provider itself (ADR 0019 §5 sanctioned form) — the fix touches the bootstrap, not the read.
- The Brain root resolves `neo.mjs` through its own `node_modules` (checkout: the shared-engine symlink; staged organism: the manifest's pinned install), so bare `neo.mjs/src/…` specifiers resolve from `cwd: repoRoot` in both shapes.
- `dotenv/config` in the daemon's bootstrap loads the Brain root's `.env`; the resolver must load the same file, or a checkout that declares its plane in `.env` resolves a different topology than the daemon it then supervises.

## The Fix

`resolveBrainPaths`'s script mirrors the daemon's bootstrap: `import 'dotenv/config'` first, then `neo.mjs/src/Neo.mjs`, `neo.mjs/src/core/_export.mjs`, `neo.mjs/src/manager/Instance.mjs`, then `./ai/config.mjs`. `brain.spec.mjs` gains a `resolveBrainPaths` arm with an injected `execFileFn` that asserts the script's import lines (red on `dev`: the relative `./src/Neo.mjs` line) and a live arm that runs the real script against the Brain root named by `NEO_AGENTOS_RUNTIME_ROOT` when set (skipped otherwise), asserting the returned leaf set. `harness/README.md`'s Run section states the Brain-root requirement in the post-split shape.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `resolveBrainPaths` script (`harness/brain.mjs`) | the Brain's own entrypoint bootstrap (`ai/daemons/orchestrator/daemon.mjs:17-27`) | bootstraps `dotenv/config` + `neo.mjs/src/*` from the Brain root's `node_modules`; leaf set unchanged | unchanged: a resolver failure rejects with the Brain's own `--migrate-config` guidance | `harness/README.md` Run | spec arms above; a live `start:brain` printing `HARNESS_BRAIN_PLAN` on this machine |

## Decision Record impact

`aligned-with ADR 0019` (config read through the Provider at the use site; no re-derivation, no env read outside the Provider) · `aligned-with ADR 0034` (the launcher gates the Brain leg on an explicit Brain root).

## Acceptance Criteria

- [ ] Red-first: a spec arm captures the resolver script through `execFileFn` and fails on `dev` for the `./src/Neo.mjs` import; green with the package imports + `dotenv/config`.
- [ ] Live arm: with `NEO_AGENTOS_RUNTIME_ROOT` set to a Brain checkout, `resolveBrainPaths` resolves `{backupPath, chromaDataDir, chromaPort, dbPath, fleetInstanceRoot, fleetPlaneBase, orchestratorDataDir}`; skipped (not silently green) without the root.
- [ ] `npm run start:brain` on this machine prints a `HARNESS_BRAIN_PLAN` line (the plan itself is the plane-attach leaf's concern, not this ticket's).
- [ ] The full `test/playwright/unit/harness/` tree stays green.
- [ ] `harness/README.md` names the post-split Brain-root shape.

## Out of Scope

- The plane-attach configuration for the host (planeBase, bearer file, the one canonical Neural Link bridge) — Vega's slice under #7.
- The packaged product's plane declaration without env (a config surface; #12's first-run card).
- The pack stage's failing `npm install` (engine `prepare` inside a git-dependency build — neomjs/neo ticket, filed 2026-09-25) and the UI-only smoke's exit code (Vega's defect-note).

## Related

#7 (Electron shell epic) · #12 (native shell UX) · #4, #115, #43 (the launcher's Brain-root history) · neomjs/neo-agent-brain `ai/daemons/orchestrator/daemon.mjs`

Live latest-open sweep: checked all 16 open issues at 2026-09-25 10:13Z; no equivalent. A2A claim sweep: Vega's 10:01Z `[FM help]` claim covers plane-attach + the shared bridge, not the resolver (split agreed by DM). Memory Core sweep: no prior decision on the resolver's bootstrap; Emmy's 2026-08-27 receiver audit named the pack's monorepo assumptions, not this import. Own-assignment sweep: none on this surface.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "harness resolveBrainPaths ./src/Neo.mjs post-split Brain root neo.mjs package bootstrap"

## Timeline

- 2026-09-25T10:17:59Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T10:18:01Z @neo-fable-clio added the `bug` label
- 2026-09-25T10:18:01Z @neo-fable-clio added the `agent-os` label
- 2026-09-25T10:18:01Z @neo-fable-clio added the `ai` label
- 2026-09-25T10:18:01Z @neo-fable-clio added the `build` label
- 2026-09-25T10:18:33Z @neo-fable-clio added parent issue #7
- 2026-09-25T10:29:35Z @neo-fable-clio cross-referenced by PR #192
- 2026-09-25T10:37:47Z @neo-fable-clio cross-referenced by #193
- 2026-09-25T10:53:51Z @tobiu referenced in commit `491ea27` - "Merge pull request #192 from neomjs/fix/191-brain-resolver-bootstrap

fix(harness): the Brain resolver bootstraps the Engine from the neo.mjs package (#191)"
- 2026-09-25T10:53:51Z @tobiu closed this issue
- 2026-09-25T10:54:24Z @neo-fable-clio cross-referenced by #195

