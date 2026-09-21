---
id: 192
title: Establish Host and Cloud package boundaries
state: CLOSED
labels:
  - epic
  - ai
  - refactoring
  - architecture
  - build
  - agent-os
assignees: []
createdAt: '2026-08-27T15:01:36Z'
updatedAt: '2026-08-28T22:22:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/192'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: null
subIssues:
  - '[x] 197 Establish deploy/host and independent deploy/cloud packages'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-28T22:22:05Z'
---
# Establish Host and Cloud package boundaries

## Context

Parent goal: neomjs/neo-agent-brain#189. The accepted product boundary is a Host-Edge root package plus an independently installed Container-Cloud package, with no npm workspaces or ancestor-hoist dependency. The live repository still mixes both planes under `ai/**` and recreates Engine-root projections during `prepare`.

This is an owner-sized lane because package manifests, deployment layout, Engine dependency imports, fresh-clone behavior, and plane isolation must converge together. No single PR can both establish the final packages and remove every compatibility projection safely.

The mandatory structure map ran successfully on a fresh Brain `dev` clone.

## Problem Scope

There is no `src/**`. Deployment definitions mix three Host files with fifteen Cloud files under `ai/deploy/**`. Root `package.json` exposes Cloud, durable-store, diagnostic, and maintenance operations. `prepare` symlinks Engine `apps`, `examples`, `harness`, `resources`, and `src`, then copies `buildScripts`; 215 source files and 483 tests depend on the projected Engine root.

That installation shape is a local reconstruction of the old monorepo, not an independent Brain package.

## Intended Solution Shape

Root becomes the Host-Edge package and exposes Host-Edge operations only. Deployment definitions live under `deploy/host/**` and `deploy/cloud/**`; `deploy/cloud/package.json` owns an independent Cloud install and Cloud entrypoints. Workspaces and hoisted dependency access remain forbidden.

Surviving Brain modules import Engine primitives through the `neo.mjs` package. Temporary Engine-root projections and sibling-checkout assumptions disappear. Source placement follows explicit Brain domains rather than the legacy `ai/**` tree.

This lane changes the real install/runtime boundary. A package census, isolation ledger, or path manifest is evidence only and cannot satisfy it.

## Out of Scope

- Internal simplification of Dream, embedding, Memory Core, or Knowledge Base behavior.
- New deployment features or topology beyond the settled Host/Cloud split.
- Engine release work.
- Preserving old paths as permanent aliases.

## Avoided Traps

- `deploy/` plus `cloud/deploy/` or another half-nested layout.
- npm workspaces or ancestor dependency hoisting.
- Renaming `ai/**` to `src/**` without domain ownership.
- Keeping root scripts as forwarding wrappers into Cloud.

## Related

Parent: neomjs/neo-agent-brain#189

Related: neomjs/neo-agent-brain#12 · neomjs/neo-agent-brain#90 · neomjs/neo-agent-brain#184

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: `Brain Host Edge root deploy host deploy cloud nested package remove Engine projections neo.mjs imports`


## Timeline

- 2026-08-27T15:01:38Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:38Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:39Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:39Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:01:39Z @neo-gpt-emmy added the `build` label
- 2026-08-27T15:01:39Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:06:39Z @neo-gpt-emmy cross-referenced by #197
- 2026-08-27T15:06:41Z @neo-gpt-emmy cross-referenced by #198
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T11:00:18Z @tobiu cross-referenced by PR #205
- 2026-08-28T22:14:39Z @neo-gpt-emmy cross-referenced by #213
### @neo-gpt-emmy - 2026-08-28T22:22:05Z

Superseded by #213. Host and Cloud are executable dependency profiles over one canonical source tree, not package-owned source planes. Closed #197 is corrected by #214; no live child remains here.

- 2026-08-28T22:22:06Z @neo-gpt-emmy closed this issue

