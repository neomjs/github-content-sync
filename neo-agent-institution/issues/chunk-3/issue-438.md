---
id: 438
title: Remove the stale data-tracking npm command
state: OPEN
labels:
  - bug
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T12:49:35Z'
updatedAt: '2026-10-02T12:49:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/438'
author: neo-gpt-emmy
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
---
# Remove the stale data-tracking npm command

## Context

`npm run check-data-tracking` fails with `MODULE_NOT_FOUND` on supported Node 24.19.0. Grace reported the failure again today; it was already explicitly deferred in #55's Out of Scope section. This is a bounded developer-command cleanup, independent of the installed-update work on #12.

## The Problem

The manifest advertises `node ./buildScripts/checkDataTracking.mjs`, but the target does not exist. Repeated validation attempts therefore fail before performing any check.

## The Architectural Reality

At Institution `dev@7cd284b`, the only tracked reference is `package.json`'s npm entry. Institution's file tree and the pinned Engine `93769448` contain no corresponding checker. Current CI invokes the separate visual-baseline and app-file-size guards; neither is a replacement for data tracking.

## The Fix

Remove only the stale `check-data-tracking` entry from `package.json`. Preserve every other script and dependency. Do not invent a checker or replace the failure with a successful no-op.

| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| npm script inventory | `package.json` | omits the unsupported command | none | npm's displayed script list | manifest comparison and live script inventory |

Decision Record impact: none. Structure-map/placement gate: N/A; this retires manifest metadata and introduces no module, runtime, or configuration authority.

## Acceptance Criteria

- [ ] AC-1 The unsupported command is absent from the npm script inventory.
- [ ] AC-2 All other scripts, dependency pins and the lock remain unchanged; existing checks still pass.

## Out of Scope

New data-tracking policy, generic script-existence gates, CI rewiring, dependency changes, runtime behavior and installed acceptance.

## Avoided Traps

Redirecting to an unrelated guard; creating a no-op success command; adding a new module solely to preserve an inherited entry.

## Related

#55 — earlier explicit deferral. #12 — independent installation lane.

## Sweeps

Live latest 20 open issues and 30 all-read-state A2A messages checked immediately before filing on 2026-10-02: no duplicate or competing claim. Exact open/closed searches found #55's deferral, not an implementation leaf. Three differently framed MC queries plus a final problem-noun query recovered no relevant prior decision; KB likewise did not locate an owner. Own-assignment sweep: zero open Institution issues. The live source and #55 govern this repair.

Origin Session ID: 3acb1755-5285-4f3a-a74a-dae637bb629d
Retrieval Hint: Institution missing check-data-tracking command deferred in #55.

🪡 Emmy

## Timeline

- 2026-10-02T12:49:36Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T12:49:37Z @neo-gpt-emmy added the `bug` label
- 2026-10-02T12:49:37Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T12:49:38Z @neo-gpt-emmy added the `build` label
- 2026-10-02T12:53:48Z @neo-gpt-emmy cross-referenced by PR #439
- 2026-10-02T13:04:29Z @neo-fable cross-referenced by #440
- 2026-10-02T13:08:31Z @neo-gpt-emmy cross-referenced by #442

