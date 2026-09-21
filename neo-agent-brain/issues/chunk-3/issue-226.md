---
id: 226
title: Complete the source cut across preflight-blocked files
state: CLOSED
labels:
  - documentation
  - dependencies
  - ai
  - refactoring
  - testing
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T03:19:11Z'
updatedAt: '2026-08-29T10:26:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/226'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 225
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T10:26:21Z'
---
# Complete the source cut across preflight-blocked files

Child of #225.

## Problem

The package-import cut reaches 13 files whose unchanged durable comments contain 33 decay-prone ticket, PR, Epic, or ADR references. The mandatory archaeology gate scans every touched file in full, so those imports cannot land while the comments remain. Folding silent provenance deletion into #225 was rejected as an unscoped cleanup.

## Scope

Rewrite only those 33 comments/JSDoc lines to state the enduring mechanism without tracking references, and land the package-qualified Engine imports in the same 13 files. No runtime or test behavior changes.

## Acceptance criteria

- [ ] The 13 named files carry no archaeology-gate findings.
- [ ] Their comments retain the behavioral rationale while dropping historical tracking coordinates.
- [ ] Their Engine imports resolve through `neo.mjs/src/**`.
- [ ] Check-only agent preflight passes the 13-file set.
- [ ] The #225 whole-branch census reaches zero relative root-src imports.

## Timeline

- 2026-08-29T03:19:11Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T03:19:12Z @neo-gpt-emmy added the `documentation` label
- 2026-08-29T03:19:13Z @neo-gpt-emmy added the `dependencies` label
- 2026-08-29T03:19:13Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T03:19:13Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-29T03:19:13Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T03:19:13Z @neo-gpt-emmy added the `build` label
- 2026-08-29T03:19:14Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T03:22:47Z @neo-gpt-emmy cross-referenced by PR #227
- 2026-08-29T10:26:21Z @tobiu closed this issue

