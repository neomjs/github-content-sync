---
id: 194
title: Make the retained Brain test suite real
state: OPEN
labels:
  - epic
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
assignees: []
createdAt: '2026-08-27T15:01:40Z'
updatedAt: '2026-08-28T22:36:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/194'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 212
subIssues:
  - '[x] 89 unit-brain: order-dependent pollution — allowlisted config mutation (opting out of working isolation) + destroy-before-initAsync lifecycle leak'
  - '[ ] 201 Run the retained Brain unit suite in CI'
  - '[x] 218 Brain unit setup names nonexistent install artifacts'
  - '[x] 257 Receive ADR-0019 guards into Brain before the Engine re-pin'
  - '[x] 420 Sync spec''s task-state double records clocks the real writer never keeps'
subIssuesCompleted: 4
subIssuesTotal: 5
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Make the retained Brain test suite real

## Problem scope

The retained Brain test corpus is not represented honestly by CI. The current Brain Unit workflow installs the required tier, lists the collection, and then executes three named smoke specs. The configuration itself correctly fails CI when that tier is absent and skips it loudly for a local base install; the coverage gap is the workflow's final run command, not a silent config omission. Large suites also preserve one-shot scripts and shared-singleton mutation after their production value has ended.

The issue is ownership and execution, not the absence of a Cloud runner. Unit and integration tests execute on Host CI. Integration tests create and exercise Docker containers as their systems under test.

## Intended solution shape

Make tests follow retained domain behavior. A domain slice deletes tests for deleted production machinery, moves focused tests with retained contracts, and removes fixtures or helpers that no longer shorten understanding.

Host CI discovers and executes the retained unit corpus without path-based silent skips. Integration suites remain Host processes that provision bounded containers, wait for readiness, exercise public boundaries, and clean up. CI scope names must report what actually ran.

This Epic changes real test ownership and reach. A coverage ledger, a second runner, or another inventory does not satisfy it.

## Out of scope

- a Cloud CI runner or Cloud-only witness tree;
- wiring every legacy spec into CI before deleting invalid coverage;
- a generic test orchestration framework;
- file-size lint as a substitute for production refactoring.

## Avoided traps

- counting specs rather than executing retained behavior;
- preserving production debt because tests surround it;
- splitting giant specs while retaining shared-state pollution;
- calling a smoke subset “the unit suite.”

## Related

Parent: #212. #201 owns the immediate unit-workflow reach defect. #89 remains a focused isolation lane.


## Timeline

- 2026-08-27T15:01:42Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:43Z @neo-gpt-emmy added the `testing` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:01:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:01:53Z @neo-gpt-emmy added parent issue #189
- 2026-08-27T15:02:15Z @neo-gpt-emmy added sub-issue #89
- 2026-08-27T15:02:15Z @neo-gpt-emmy added sub-issue #17
- 2026-08-27T15:06:46Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-27T15:07:13Z @neo-gpt-emmy added sub-issue #201
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T11:00:18Z @tobiu cross-referenced by PR #205
- 2026-08-28T22:17:33Z @neo-gpt-emmy removed parent issue #189
- 2026-08-28T22:17:34Z @neo-gpt-emmy added parent issue #212
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #193
- 2026-08-28T22:23:14Z @neo-gpt-emmy removed sub-issue #17
- 2026-08-28T22:23:25Z @neo-gpt-emmy cross-referenced by #17
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T22:31:05Z @neo-gpt-emmy cross-referenced by #15
- 2026-08-28T22:31:05Z @neo-gpt-emmy cross-referenced by #91
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #59
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #134
- 2026-08-28T23:06:50Z @neo-gpt-emmy cross-referenced by #218
- 2026-08-28T23:07:00Z @neo-gpt-emmy added sub-issue #218
- 2026-08-28T23:12:38Z @neo-gpt-emmy cross-referenced by PR #219
- 2026-08-29T10:12:12Z @neo-fable cross-referenced by #17844
- 2026-08-30T17:02:58Z @neo-opus-ada cross-referenced by PR #254
- 2026-08-30T17:45:56Z @neo-gpt-emmy cross-referenced by PR #255
- 2026-08-30T19:28:07Z @neo-gpt-emmy cross-referenced by #257
- 2026-08-30T19:28:17Z @neo-gpt-emmy added sub-issue #257
- 2026-08-30T20:36:43Z @neo-opus-grace cross-referenced by PR #259
- 2026-08-30T21:33:37Z @neo-opus-ada cross-referenced by PR #264
- 2026-08-31T03:14:12Z @neo-opus-grace cross-referenced by #271
- 2026-09-02T11:30:42Z @neo-fable cross-referenced by PR #299
- 2026-09-21T10:58:41Z @neo-opus-ada cross-referenced by PR #396
- 2026-09-23T01:41:10Z @neo-opus-vega cross-referenced by #420
- 2026-09-23T01:41:35Z @neo-opus-vega added sub-issue #420

