---
id: 345
title: Patch the runtime gray-matter js-yaml merge vulnerability
state: CLOSED
labels:
  - bug
  - dependencies
  - ai
  - security
assignees:
  - neo-gpt
createdAt: '2026-09-12T11:39:46Z'
updatedAt: '2026-09-12T12:59:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/345'
author: neo-gpt
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
blocking: []
closedAt: '2026-09-12T12:59:16Z'
---
# Patch the runtime gray-matter js-yaml merge vulnerability

## Context

The operator requested a dependency security sweep on 2026-09-12. [GHSA-2883-xcg3-v3hh](https://github.com/advisories/GHSA-2883-xcg3-v3hh) affects js-yaml 3.x below 3.15.2: empty merge sources consume CPU without consuming the merge budget.

## The Problem

The root override permits `gray-matter -> js-yaml ^3.15.1`, and the current lock resolves 3.15.1. The direct js-yaml 5.x dependency does not replace this nested copy.

## The Architectural Reality

`gray-matter@4.0.3` uses the 3.x safeLoad/safeDump API. The existing scoped override in package.json owns its resolution; a global major override would change that contract. This is a dependency repair with no source/API or ADR change.

## The Fix

Raise only `overrides.gray-matter.js-yaml` to `^3.15.2` and regenerate package-lock.json. Preserve the direct js-yaml dependency and unrelated resolved packages.

## Acceptance Criteria

- [ ] The scoped override requires >=3.15.2 and the nested lock/install resolves a patched 3.x version.
- [ ] An audit no longer reports GHSA-2883-xcg3-v3hh for this lock; remaining unrelated findings are disclosed.
- [ ] gray-matter front-matter parsing/stringifying succeeds and direct js-yaml remains on 5.x.
- [ ] The diff contains only the manifest floor and necessary lock change; repository checks pass.

## Out of Scope

Other dependency upgrades, replacing gray-matter, alert dismissals, and production deployment.

## Avoided Traps

Do not override the nested package to 5.x or treat the direct 5.x resolution as proof about gray-matter. Do not run a broad forced audit upgrade.

Decision Record impact: none.

Origin Session ID: 92f5d790-2865-4f69-b25e-150175745a6d

Creation freshness: latest 20 open issues per repository, all-state YAML searches, own assignments and 30 recent all-status A2A messages checked immediately before filing; no equivalent or competing claim. The Memory Core problem query returned unrelated history; live manifest/advisory evidence determines this ticket.

## Related

Brain's GitHub alert API returns an empty set, but dev's package-lock.json contains the affected runtime dependency. Sharp remediation remains in #300; it is independent of this compatible patch.

## Timeline

- 2026-09-12T11:39:46Z @neo-gpt assigned to @neo-gpt
- 2026-09-12T11:39:47Z @neo-gpt added the `bug` label
- 2026-09-12T11:39:47Z @neo-gpt added the `dependencies` label
- 2026-09-12T11:39:47Z @neo-gpt added the `ai` label
- 2026-09-12T11:39:48Z @neo-gpt added the `security` label
- 2026-09-12T11:40:33Z @neo-gpt cross-referenced by #300
- 2026-09-12T11:52:15Z @neo-gpt cross-referenced by PR #346
- 2026-09-12T12:59:16Z @tobiu referenced in commit `0fde4ae` - "Merge pull request #346 from neomjs/codex/345-yaml-security

fix(deps): bound gray-matter empty-merge parsing (#345)"
- 2026-09-12T12:59:16Z @tobiu closed this issue

