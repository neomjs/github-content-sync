---
id: 225
title: Retire the Engine src projection
state: CLOSED
labels:
  - bug
  - dependencies
  - ai
  - refactoring
  - testing
  - architecture
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T02:59:22Z'
updatedAt: '2026-08-29T10:26:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/225'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 198
subIssues:
  - '[x] 226 Complete the source cut across preflight-blocked files'
subIssuesCompleted: 1
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T10:26:21Z'
---
# Retire the Engine src projection

Child of #198.

## Problem

Brain `src` is an install-created symlink to `node_modules/neo.mjs/src`. It masks 1,423 relative Engine-source imports and makes the canonical Brain source root impossible: a move into `src/**` writes into the dependency while Git records only deletion.

## Scope

Rewrite retained Engine primitive imports to package-qualified `neo.mjs/src/**` specifiers across Brain source and tests. Remove `src` from the Engine projection materializer. Keep the script-plane closure sound across this first-party package boundary by following only pinned `neo.mjs/src/**` modules; arbitrary bare packages remain leaves. Refresh formatting-sensitive baseline identity only where the required import alignment changes its exact source text, without adding debt authority.

Do not create an alias or move Brain production code in this cut.

## Acceptance criteria

- [ ] Tracked JavaScript contains zero relative import specifiers into a repository-root `src/**` projection.
- [ ] Every `neo.mjs/src/**` specifier resolves through the installed package.
- [ ] The full unit collection succeeds while the repository-root `src` symlink is absent.
- [ ] `npm prepare` no longer creates the `src` projection.
- [ ] Script-plane closure follows `neo.mjs/src/**` far enough to preserve inherited-member and capability analysis, while an unrelated bare package remains a leaf.
- [ ] Script-plane and AiConfig lints remain green with no new unresolved-edge or module-capture baseline rows.
- [ ] No compatibility alias, copied Engine source, or sibling-checkout fallback is introduced.

## Evidence boundary

Exact pre-change census: 1,423 imports across 702 files. The other Engine projections (`apps`, `examples`, `harness`, `resources`, `buildScripts`) remain owned by parent #198 and are explicitly out of this review.

## Timeline

- 2026-08-29T02:59:23Z @neo-gpt-emmy added the `bug` label
- 2026-08-29T02:59:23Z @neo-gpt-emmy added the `dependencies` label
- 2026-08-29T02:59:23Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T02:59:24Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T02:59:24Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-29T02:59:24Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T02:59:24Z @neo-gpt-emmy added the `architecture` label
- 2026-08-29T02:59:24Z @neo-gpt-emmy added the `build` label
- 2026-08-29T02:59:24Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T03:19:12Z @neo-gpt-emmy cross-referenced by #226
- 2026-08-29T03:22:47Z @neo-gpt-emmy cross-referenced by PR #227
- 2026-08-29T03:29:37Z @tobiu referenced in commit `659217a` - "fix(lint): follow pinned Engine package source (#225)"
- 2026-08-29T10:26:21Z @tobiu referenced in commit `82862bb` - "Merge pull request #227 from neomjs/codex/198-remove-engine-projections

fix(dependencies): retire the Engine src projection (#225)"
- 2026-08-29T10:26:21Z @tobiu closed this issue
- 2026-08-29T10:46:33Z @neo-gpt-emmy cross-referenced by #231
- 2026-08-29T10:49:24Z @neo-gpt-emmy cross-referenced by PR #232
- 2026-08-29T16:22:07Z @neo-gpt-emmy cross-referenced by #215
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324

