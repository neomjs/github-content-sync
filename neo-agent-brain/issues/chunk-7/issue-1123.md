---
id: 1123
title: 'Retire the Brain''s front-door fingerprint guard: neo owns it again'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:14:12Z'
updatedAt: '2026-10-19T02:52:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1123'
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
blocking: []
closedAt: '2026-10-19T02:52:38Z'
---
# Retire the Brain's front-door fingerprint guard: neo owns it again

## Context

This is a #201 single: `test/playwright/unit/ai/scripts/lint/checkFrontDoorFingerprint.spec.mjs` › `the LIVE front door passes its own fingerprint check` fails with `ENOENT` on `learn/benefits/Introduction.md`.

## The Problem

`ai/scripts/lint/check-front-door-fingerprint.mjs` guards neo's front door, a file the Brain does not have. No Brain workflow runs the guard, and its live arm can only fail here. Meanwhile neo's front door drifted unguarded: on neo `dev` it declares 55,414 bytes and has 56,728.

## The Architectural Reality

- neomjs/neo#19853 returns the guard to neo, as `buildScripts/util/check-front-door-fingerprint.mjs` with its spec and its path-filtered workflow, under ADR 0040 §2.7: "Engine-only contributor guards with no Brain dependency stay Engine-owned." Its PR is neomjs/neo#19854.
- Nothing in the Brain references the script or its spec, outside the two files themselves.

## The Fix

Delete `ai/scripts/lint/check-front-door-fingerprint.mjs` and its spec. This lands after neomjs/neo#19854, so the guard exists in exactly one place.

## Acceptance Criteria

- [ ] **AC-1:** the Brain carries neither file, and nothing in the Brain references them.
- [ ] **AC-2:** the unit suite no longer lists `the LIVE front door passes its own fingerprint check`.

## Out of Scope

- The other callerless `pin-fetch` gates that neomjs/neo#19853 lists.

## Related

- #201: the burn-down.
- neomjs/neo#19853, neomjs/neo#19854: the guard's return.
- neomjs/neo#17783: the R3 precedent.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T07:00:13Z and rechecked for this title; no equivalent found.
A2A claim sweep: the latest 20 inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119, #1121.
Structure map: N/A; this deletes two files.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "retire Brain check-front-door-fingerprint; guard returned to neo buildScripts/util; Introduction.md ENOENT in Brain unit suite"

## Timeline

- 2026-10-18T07:14:12Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:14:13Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:14:13Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:14:13Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:14:50Z @neo-opus-ada cross-referenced by PR #1124
- 2026-10-18T07:20:59Z @neo-opus-ada cross-referenced by #1125
- 2026-10-18T07:24:28Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T07:27:25Z @neo-opus-ada cross-referenced by #1127
- 2026-10-18T07:33:14Z @neo-opus-ada cross-referenced by #1129
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131
- 2026-10-19T02:52:39Z @tobiu closed this issue
- 2026-10-19T02:52:39Z @tobiu referenced in commit `f069887` - "fix(test): the front-door fingerprint guard leaves the Brain for neo (#1123) (#1124)

The guard checks neo's learn/benefits/Introduction.md, which the Brain does not carry; no Brain workflow ran it, and its live arm could only fail here. neomjs/neo#19854 returns it to neo's buildScripts/util with its workflow."

