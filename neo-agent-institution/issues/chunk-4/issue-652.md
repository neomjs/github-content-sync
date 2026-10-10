---
id: 652
title: Correct the FM quit guidance for detached desktop harnesses
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-09T22:52:33Z'
updatedAt: '2026-10-09T23:09:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/652'
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
closedAt: '2026-10-09T23:09:51Z'
---
# Correct the FM quit guidance for detached desktop harnesses

## Context

During the installed update, FM exited but detached desktop harnesses survived. The operator explicitly corrected the intended contract: restarting FM must not close or kill those harnesses, then requested the README correction.

## The Problem

`harness/README.md` (permission and update sections), the header of `harness/install.mjs`, and `PEER_QUIT_WARNING` falsely say that quitting FM also stops its peer harnesses. That statement led the update procedure to expect the wrong shutdown behavior.

## The Architectural Reality

Design authority: the operator's October 10 clarification, corroborated by Brain `FleetLifecycleService`'s detached app-bundle launch and lease/re-adoption contract. Explicit per-seat Stop is separate from FM shell teardown. CLI families retain their own lifecycle rules; do not generalize desktop detachment to every harness family.

The installer's bundle-process census is another boundary: local MCP executables/scripts may still reside inside the installed bundle even while they connect to an external Agent OS. An installer refusal because those files remain in use is not evidence that FM should own the detached harness lifetime.

## The Fix

Correct the README, matching installer explanatory comments, and the human-readable warning. Explain ordinary FM restart versus the current bundle-use limitation during replacement. Preserve the existing process census, shutdown implementation, custody checks and rollback behavior.

## Acceptance Criteria

- [ ] README states that provisioned detached desktop harnesses survive FM quit/restart; explicit seat Stop/Restart is a distinct action.
- [ ] Installer comments and warning make the same distinction without claiming quit stops those seats.
- [ ] Current bundle-file use is distinguished from remote Agent OS connectivity; no claim that an in-use replacement is already supported.
- [ ] No process-control, runtime provisioning, profile, credential or installer-guard behavior changes.

## Out of Scope

Decoupling/versioning the MCP runtime for uninterrupted bundle updates; changing CLI harness lifetime; the installed update or its Electron drag acceptance.

Decision Record impact: none — correct descriptive guidance to the existing detached-desktop contract. Structure map: N/A, documentation and warning text only; no placement change or new module. Contract ledger: no API, flags or result shape change.

## Related

#12, #354, #473, neomjs/neo-agent-brain#660.

Live latest-open sweep: latest 20 created-descending open Institution issues checked immediately before filing; no equivalent. Recent 30 all-state A2A messages contain the incident note, no competing correction claim. Historical search found closed #354/#473, not an open correction. MC query on FM quit/detached lifetime recovered Brain #660's survival precedent (memory f6aa2326-dff7-4415-a791-125494e9f54e). Own-assignment sweep found only #42, unrelated.

Origin Session ID: b56dbc41-6e95-4210-a2ea-8d1f5f3ffcd0
Retrieval Hint: "FM quit detached desktop harness survival README bundle runtime dependency"

## Timeline

- 2026-10-09T22:52:33Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-09T22:52:34Z @neo-gpt-emmy added the `bug` label
- 2026-10-09T22:52:34Z @neo-gpt-emmy added the `documentation` label
- 2026-10-09T22:52:35Z @neo-gpt-emmy added the `ai` label
- 2026-10-09T22:56:27Z @neo-gpt-emmy cross-referenced by PR #653
- 2026-10-09T23:03:29Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-09T23:09:51Z @tobiu referenced in commit `af1b85d` - "docs(harness): correct detached seat quit guidance (#652) (#653)"
- 2026-10-09T23:09:52Z @tobiu closed this issue

