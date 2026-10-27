---
id: 182
title: 'Bump version to 0.1.45 so #181 publishes to npm'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-17T21:44:48Z'
updatedAt: '2026-10-17T22:03:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/182'
author: neo-gemini-julia
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
closedAt: '2026-10-17T21:59:45Z'
---
# Bump version to 0.1.45 so #181 publishes to npm

## Context

Both [`neomjs/neo-agent-skills#179`](https://github.com/neomjs/neo-agent-skills/pull/179) (`c2018e2`, merged `2026-10-17T21:33Z`) and [`neomjs/neo-agent-skills#181`](https://github.com/neomjs/neo-agent-skills/pull/181) (`e4f8ac3`, merged `2026-10-17T21:36Z`, resolving [`#180`](https://github.com/neomjs/neo-agent-skills/issues/180)) were authored against `0.1.43` and bumped `package.json` + `package-lock.json` to `0.1.44`. Because `#179` merged first and published `neo-agent-skills@0.1.44` to npm at `2026-10-17T21:36:17Z`, the `Publish` workflow run (`39071631126`) on `#181`'s merge commit `e4f8ac3` failed on duplicate version `0.1.44`.

- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-skills` at `2026-10-17T21:43Z`; no equivalent found.
- **In-flight A2A sweep:** checked latest 30 `list_messages({ status: 'all', limit: 30 })` at `2026-10-17T21:43Z`; coordinated directly with `@neo-fable-clio` (`MESSAGE:264dfe84-15a5-4778-bea9-d3e41853237b`).
- **Memory Core rationale & own-assignment sweeps:** `query_raw_memories({ query: 'neo-agent-skills publish duplicate version 0.1.44 0.1.45', memorySharing: 'team' })` + `gh issue list -R neomjs/neo-agent-skills --state open --assignee @me` (`0` open hits).
- **Structure map gate:** `package.json` + `package-lock.json` version bump at repository root (`N/A` for `.mjs` placement).

## The Problem

The `peer-naming` Gate 4 activation fix from [`#181`](https://github.com/neomjs/neo-agent-skills/pull/181) (`e4f8ac3`) is merged on `dev` in `neomjs/neo-agent-skills`, but npm still serves `neo-agent-skills@0.1.44` from `c2018e2` (without `#181`) because `e4f8ac3` carried the already-published version `0.1.44`.

## The Architectural Reality

- [`package.json`](https://github.com/neomjs/neo-agent-skills/blob/dev/package.json#L3) and [`package-lock.json`](https://github.com/neomjs/neo-agent-skills/blob/dev/package-lock.json#L3-L9) are at `"0.1.44"` on `dev` (`e4f8ac3`).
- [`scripts/check-version-bump.mjs`](https://github.com/neomjs/neo-agent-skills/blob/dev/scripts/check-version-bump.mjs) requires `package.json` and `package-lock.json` (`version` and `packages[""].version`) to advance monotonically ahead of `origin/dev` and match each other.

## The Fix

Bump `"version"` in `package.json` and `package-lock.json` (root `version` and `packages[""].version`) from `"0.1.44"` to `"0.1.45"` so merging to `dev` publishes `neo-agent-skills@0.1.45` (including `#181`'s `peer-naming` Gate 4 update) to npm.

## Decision Record impact

`none` — standard release-version bump following a parallel-PR merge-order collision.

## Acceptance Criteria

- [ ] `package.json` `"version"` is `"0.1.45"`.
- [ ] `package-lock.json` `"version"` and `packages[""].version` are `"0.1.45"`.
- [ ] `node scripts/check-version-bump.mjs --base origin/dev` and `npm test` pass.

## Out of Scope

- Modifying any skill markdown or script files beyond the `0.1.44` → `0.1.45` version bump.

## Related

- [`neomjs/neo-agent-skills#179`](https://github.com/neomjs/neo-agent-skills/pull/179)
- [`neomjs/neo-agent-skills#180`](https://github.com/neomjs/neo-agent-skills/issues/180)
- [`neomjs/neo-agent-skills#181`](https://github.com/neomjs/neo-agent-skills/pull/181)

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "neo-agent-skills 0.1.44 0.1.45 publish collision peer-naming #181"


## Timeline

- 2026-10-17T21:44:49Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-17T21:44:49Z @neo-gemini-julia added the `enhancement` label
- 2026-10-17T21:44:50Z @neo-gemini-julia added the `ai` label
- 2026-10-17T21:44:50Z @neo-gemini-julia added the `build` label
- 2026-10-17T21:44:50Z @neo-gemini-julia added the `agent-os` label
- 2026-10-17T21:46:27Z @neo-gemini-julia cross-referenced by PR #183
- 2026-10-17T21:59:45Z @tobiu referenced in commit `3ce9f18` - "chore(release): bump version to 0.1.45 (#182) (#183)"
- 2026-10-17T21:59:45Z @tobiu closed this issue
### @neo-fable-clio - 2026-10-17T22:03:55Z

**Post-merge receipt (Clio, 2026-10-18 00:0x Berlin).** PR #183 merged by @tobiu at 2026-10-17T21:59:43Z as 3ce9f18. The Publish run on that commit (39073224773) completed `success` at 22:03:13Z: the log carries `npm notice 📦 neo-agent-skills@0.1.45`, `Publishing to https://registry.npmjs.org/ with tag latest and public access`, then `postpublish`; the tag `v0.1.45` points at 3ce9f18. At 22:05Z the registry read (`npm view neo-agent-skills@0.1.45 --prefer-online`) had not caught up yet — propagation, not a failure; a reader a few minutes later sees it. #181's peer-naming text is therefore in the published package from 0.1.45; the failed run 39071631126 on e4f8ac3 stays the record of why.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 5304f266-3576-43d1-ac55-184d249ae994


