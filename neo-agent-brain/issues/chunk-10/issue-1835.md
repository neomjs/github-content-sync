---
id: 1835
title: Every Brain CI install fails on onnxruntime-node's CUDA download
state: CLOSED
labels:
  - bug
  - dependencies
  - ai
  - build
assignees:
  - neo-opus-ada
createdAt: '2026-11-18T02:18:38Z'
updatedAt: '2026-11-18T02:44:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1835'
author: neo-opus-ada
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
  - '[ ] 1814 Cutover binds existing identities and retires former-login redirects'
closedAt: '2026-11-18T02:44:51Z'
---
# Every Brain CI install fails on onnxruntime-node's CUDA download

## Context

On PR #1834 at 02:13Z, 2026-11-18, three jobs failed in `npm ci` before any check ran: `lint` (Script Plane Lint, run `44221679839`), `substrate` (substrate-sync, run `44221679874`) and `Shared PR Baseline / Skills materialized` (run `44221680589`). Each failed in `node_modules/onnxruntime-node/script/install.js` with `Failed to download the binaries: 404 Not Found` for `https://github.com/microsoft/onnxruntime/releases/download/v1.21.0/onnxruntime-linux-x64-gpu-1.21.0.tgz`.

Measured at about 02:30Z: `https://github.com/microsoft/onnxruntime` answers 404, and so does `GET /repos/microsoft/onnxruntime`. `onnxruntime-node@1.21.0` is still on the npm registry (200). Unknown: why the repository is gone, and whether it comes back.

## The Problem

Brain pulls `onnxruntime-node@1.21.0` in transitively: `@chroma-core/default-embed@0.1.9` → `@huggingface/transformers@3.8.1` → `onnxruntime-node`. On Linux x64, its install script downloads the CUDA execution-provider binaries by default whenever the package lacks them. So every CI job that runs `npm ci` with install scripts now fails until that repository returns.

Brain never uses the CUDA provider. No `ai/` module selects a CUDA or GPU device: `grep -rnE "device\s*:\s*['\"](cuda|gpu)"` over `ai/` finds nothing. The CPU binaries ship inside the npm package. The download was dead weight in every Linux install even before it started failing.

The engine repository is unaffected: its `package-lock.json` holds no `onnxruntime-node`. The plane image is unaffected too: `deploy/cloud/Dockerfile` runs `npm ci --ignore-scripts`.

## The Architectural Reality

- `install.js` reads `ONNXRUNTIME_NODE_INSTALL_CUDA`, or `npm_config_onnxruntime_node_install_cuda`, and exits before any download when the value is `skip`.
- A repo-root `.npmrc` key `onnxruntime-node-install-cuda=skip` reaches install scripts as `npm_config_onnxruntime_node_install_cuda`. Probed on npm 11.21.1 with a `postinstall` that prints it: `flag=skip`. npm warns: `Unknown project config "onnxruntime-node-install-cuda". This will stop working in the next major version of npm.`
- One file covers every install site in a Brain checkout: the workflows, the shared baseline's materialization step that installs inside this checkout, and Linux developer machines. Brain has no `.npmrc` today.

## The Fix

Add `.npmrc` at the repository root with `onnxruntime-node-install-cuda=skip`, and a comment that says why the key exists and when it retires.

## Acceptance Criteria

- [ ] AC-1: `npm ci` with install scripts completes on Linux x64 CI without fetching CUDA binaries. Receipt: this PR's own `lint`, `substrate` and `Skills materialized` jobs pass.
- [ ] AC-2: CPU inference is unchanged. The Brain unit suite passes on this PR.
- [ ] AC-3: The `.npmrc` comment names its retirement trigger: npm's next major drops unknown project config (the setting then moves to `ONNXRUNTIME_NODE_INSTALL_CUDA` in the workflows), or no dependency pulls `onnxruntime-node` any more.

## Out of Scope

- Upgrading or pinning `onnxruntime-node`, `@huggingface/transformers` or `@chroma-core/default-embed`.
- Replacing the embedder.

## Avoided Traps

- **An env var in each workflow.** It misses the shared baseline's materialization step, which is defined in `neo-agent-skills` and installs inside this checkout. It also misses Linux developer machines, and it costs one edit per workflow.
- **`npm ci --ignore-scripts` everywhere.** It also skips the install hooks that substrate-sync exists to exercise.

## Related

#1834 (red on this). Live latest-open + closed sweep: the latest 20 open Brain issues, plus an open and closed search (`onnxruntime OR "npm ci" install CUDA`), at 2026-11-18T02:35Z. One closed hit, #1142, is about Fleet boot time: unrelated. No equivalent found. Band: 154 open (red); no owning ticket to fold into. A2A sweep: the latest 30 messages in all read states hold no claim on this. MC sweep (`npm ci fails in CI downloading onnxruntime-node CUDA binaries 404`): no prior decision. Own-assignment sweep: no same-surface ticket.

Owner: self-assigned, `@neo-opus-ada`.

Origin Session ID: 20309c56-00e1-4bb8-86a9-2b402554aebb

Retrieval Hint: "onnxruntime-node CUDA download 404 npm ci Brain CI"

## Timeline

- 2026-11-18T02:18:38Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-18T02:18:39Z @neo-opus-ada added the `bug` label
- 2026-11-18T02:18:40Z @neo-opus-ada added the `dependencies` label
- 2026-11-18T02:18:40Z @neo-opus-ada added the `ai` label
- 2026-11-18T02:18:40Z @neo-opus-ada added the `build` label
- 2026-11-18T02:19:28Z @neo-opus-ada cross-referenced by PR #1836
- 2026-11-18T02:20:33Z @neo-opus-ada marked this issue as blocking #1814
- 2026-11-18T02:44:51Z @tobiu referenced in commit `cbda560` - "fix(build): Brain installs skip onnxruntime-node's CUDA download (#1835) (#1836)

onnxruntime-node's Linux x64 install script fetches CUDA binaries from github.com/microsoft/onnxruntime, which answers 404, so every CI npm ci with scripts failed. Brain runs on CPU only and the CPU binaries ship in the npm package, so a repo-root .npmrc sets onnxruntime-node-install-cuda=skip for every install in a Brain checkout."
- 2026-11-18T02:44:52Z @tobiu closed this issue

