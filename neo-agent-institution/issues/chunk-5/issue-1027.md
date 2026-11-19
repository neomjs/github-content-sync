---
id: 1027
title: Every Institution npm ci fails on onnxruntime-node's CUDA download
state: CLOSED
labels:
  - bug
  - ai
  - build
  - dependencies
assignees:
  - neo-opus-vega
createdAt: '2026-11-19T20:26:33Z'
updatedAt: '2026-11-19T21:18:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1027'
author: neo-opus-vega
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
blocking:
  - '[ ] 1025 The FM installer verifies with the pinned Brain, older than the bundle'
closedAt: '2026-11-19T20:51:04Z'
---
# Every Institution npm ci fails on onnxruntime-node's CUDA download

Curated labels are not applied: this blocks every Institution PR's CI, so it is owned and urgent.

## Context

Observed: `npm ci` stops in `onnxruntime-node@1.21.0`'s postinstall with `Failed to download the binaries: 404 Not Found` for `onnxruntime-linux-x64-gpu-1.21.0.tgz` from the `microsoft/onnxruntime` releases.
- #1026: run 44604884394, job "Shared PR Baseline / Skills materialized".
- Dependabot: runs 44535363149 and 44535349096 at 13:37Z.

The asset still answers 404 at 20:10Z. The Brain hit the same failure on 11-18 (neomjs/neo-agent-brain#1835), fixed by neomjs/neo-agent-brain#1836.

## The Problem

The Institution installs `onnxruntime-node` through `neo-agent-brain` → `@chroma-core/default-embed@0.1.9` → `@huggingface/transformers@3.8.1` → `onnxruntime-node@1.21.0`. On Linux x64 its install script downloads CUDA binaries, and that download now 404s. Every CI job that runs `npm ci` with scripts fails. On #1026 it failed inside npm's git clone of `neo-agent-brain`, whose lock pin `d6469d3b` predates #1836, so the clone has no skip either.

## The Architectural Reality

- The Institution `.npmrc` carries only `foreground-scripts=true`.
- The Brain's `.npmrc` (since #1836) carries `onnxruntime-node-install-cuda=skip`, with a retirement trigger. The CPU binaries ship in the npm package, and the Fleet runs on CPU only.
- npm exports every non-default config, project `.npmrc` keys included, as `npm_config_*` into its own environment (`@npmcli/config` `set-envs.js` walks the layered config). The git dependency's prepare install inherits that environment.

## The Fix

Add `onnxruntime-node-install-cuda=skip` to the Institution's `.npmrc`, with the same comment and retirement trigger as the Brain's. If a CI install still fails inside the git clone, move the `neo-agent-brain` lock past `cbda560f` (#1836) in the same PR.

## Acceptance Criteria

- [ ] On the fix PR, "Shared PR Baseline / Skills materialized" and the Institution's own `npm ci` jobs install without the CUDA download.
- [ ] After bringing `dev` in, #1026 installs green (receipt on this ticket).

## Out of Scope

- How the Brain pin moves in general (#1025 makes the skew loud; the pin form is a separate decision).
- Upgrading `@chroma-core/default-embed` or `@huggingface/transformers` off `onnxruntime-node@1.21.0`.

## Related

- neomjs/neo-agent-brain#1835, neomjs/neo-agent-brain#1836: the same failure and fix on the Brain side.
- #1026: blocked by this.

Decision Record impact: none.
Structure map: N/A (one dotfile line).
Live latest-open + closed sweep: I read the latest 20 open issues and ran open and closed searches for "onnxruntime", "npm ci", "CUDA" and "Skills materialized" (hits #977, #657 and #676 are unrelated). The A2A claims of the last hour, an MC sweep on the symptom and my own assignments show no equivalent (2026-11-19T20:28Z). Backlog band: red (59 open); no owning ticket to fold into.

Retrieval Hint: "Institution npm ci onnxruntime-node CUDA download 404 .npmrc skip"
Origin Session ID: 9aed6db8-ae0a-4337-ab59-2720d08ab304

## Timeline

- 2026-11-19T20:26:33Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-19T20:26:34Z @neo-opus-vega added the `bug` label
- 2026-11-19T20:26:35Z @neo-opus-vega added the `ai` label
- 2026-11-19T20:26:35Z @neo-opus-vega added the `build` label
- 2026-11-19T20:26:35Z @neo-opus-vega added the `dependencies` label
- 2026-11-19T20:27:35Z @neo-opus-vega cross-referenced by PR #1028
- 2026-11-19T20:33:17Z @neo-opus-vega marked this issue as blocking #1025
- 2026-11-19T20:41:05Z @neo-opus-vega cross-referenced by #1025
- 2026-11-19T20:51:04Z @tobiu referenced in commit `647a886` - "fix(build): Institution installs skip onnxruntime-node's CUDA download (#1027) (#1028)

onnxruntime-node 1.21.0, reached through the Brain's chroma default embedder,
downloads CUDA binaries from microsoft/onnxruntime in its Linux x64 install
script, and that download answers 404. Every CI `npm ci` failed on it. The
.npmrc key the Brain carries since neomjs/neo-agent-brain#1836 skips it; npm
exports project config to its child processes, so the git-clone install of the
pinned Brain skips it too."
- 2026-11-19T20:51:05Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-19T21:18:21Z

**AC-2 receipt.** #1026 brought `dev` in after #1028 merged; at its head `07f6d0af97` the three jobs that failed on onnxruntime-node's CUDA 404 install and pass:

- [Isolated Institution](https://github.com/neomjs/neo-agent-institution/actions/runs/44614156074/job/135878245143): success
- [Explicit Brain contract](https://github.com/neomjs/neo-agent-institution/actions/runs/44614156074/job/135878245682): success
- [Shared PR Baseline / Skills materialized](https://github.com/neomjs/neo-agent-institution/actions/runs/44614156658/job/135878247232): success

At the previous head `c62ce10d` all three failed with `Failed to download the binaries: 404 Not Found` from `onnxruntime-node/script/install.js`.

— Vega (Claude Opus 5.5, Claude Code) 🌿


