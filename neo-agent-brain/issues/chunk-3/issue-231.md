---
id: 231
title: Post-cut unit collection still imports the removed src projection
state: CLOSED
labels:
  - bug
  - dependencies
  - ai
  - testing
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T10:46:32Z'
updatedAt: '2026-08-29T11:00:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/231'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 198
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T11:00:05Z'
---
# Post-cut unit collection still imports the removed src projection

Successor to merged #225; parent architecture lane #198.

## Problem

Fresh CI after the Engine `src` projection cut fails unit collection because `EmbeddingAdmission.spec.mjs`, merged through a concurrent PR, still imports two repository-root `src/**` paths. A stale ignored symlink in existing checkouts masks the defect locally.

Exact current-dev census: one file, two import specifiers, zero other relative root-`src` imports.

## Scope

Rewrite those two imports to `neo.mjs/src/**`. Add no alias, projection, fallback, or unrelated change.

## Acceptance criteria

- [ ] Current tracked JavaScript again has zero relative root-`src` import specifiers.
- [ ] The focused EmbeddingAdmission spec passes with workers=1.
- [ ] Full unit collection succeeds with repository-root `src` absent.
- [ ] The fresh hosted Brain Unit collection returns green.

## Timeline

- 2026-08-29T10:46:33Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T10:46:33Z @neo-gpt-emmy added the `bug` label
- 2026-08-29T10:46:33Z @neo-gpt-emmy added the `dependencies` label
- 2026-08-29T10:46:33Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T10:46:34Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T10:46:34Z @neo-gpt-emmy added the `build` label
- 2026-08-29T10:46:34Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T10:49:24Z @neo-gpt-emmy cross-referenced by PR #232
- 2026-08-29T11:00:05Z @tobiu referenced in commit `ea336dd` - "Merge pull request #232 from neomjs/codex/231-post-cut-unit-import

fix(test): use installed Engine after source cut (#231)"
- 2026-08-29T11:00:05Z @tobiu closed this issue
- 2026-08-29T11:37:10Z @neo-opus-vega cross-referenced by #233

