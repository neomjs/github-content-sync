---
id: 2
title: Fleet cards expose stale provider validation
state: CLOSED
labels:
  - bug
  - accessibility
  - agent-os
  - ai
  - design
  - regression
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-08-27T08:52:42Z'
updatedAt: '2026-08-28T19:58:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/2'
author: tobiu
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1 The Fleet Manager arrives with its own test suite'
blocking: []
closedAt: '2026-08-28T19:58:16Z'
---
# Fleet cards expose stale provider validation

## Context

The extraction receiver deliberately copied the Fleet card unit witness before importing any view or SCSS repair. Its stale-validation case is red in Institution: the record carries provider-validation provenance, but the card renders only the ordinary presence band.

Live latest-open sweep: checked immediately before creation at 2026-08-27T08:52:40.721Z; no equivalent Institution ticket existed.

## The Problem

`test/playwright/unit/apps/agentos/view/fleet/roster/card/container.spec.mjs` expects a `stale-validated` presence observation to render an explicit, accessible exception and then clear without residue on the next fresh observation. `apps/agentos/view/fleet/roster/card/Container.mjs` currently ignores `validationState` and `since`; the matching SCSS has no stale-validation modifier.

This was intentionally not fixed inside the custody PR: move first, repair later under Institution authority.

## The Architectural Reality

The provider owns validation provenance. `FleetAgent` passes the observation through; the card may project it but must not infer, latch, or mutate it. Text is required because color alone cannot carry the exception.

Decision Record impact: none.

## The Fix

- Extend the Fleet record's presence passthrough documentation for `validationState` and `since`.
- Project `stale-validated` in `AgentCard.syncRecord()` as visible text plus `aria-label` / `title`.
- Clear class, text, and accessibility metadata when a fresh observation replaces it.
- Add the narrow SCSS modifier without changing card geometry or unrelated goldens.
- Convert the receiver's expected-red witness back to an ordinary passing test.

## Acceptance Criteria

- [ ] A stale-validated presence renders the explicit words `validation stale`.
- [ ] The exception has accessible text and does not rely on color alone.
- [ ] A subsequent fresh observation clears every stale-validation residue.
- [ ] The focused card unit witness passes as an ordinary test.
- [ ] No unrelated Fleet visual baseline is refreshed.

## Out of Scope

- AgentCard density/layout redesign.
- Activity-stream BufferedList repairs.
- Broad visual-baseline refresh.

## Related

Related: #1

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: "Institution Fleet card stale provider validation expected-red"

Authored by Emmy (GPT-5.6 Sol Ultra, Codex). Session d39e8182-295f-418a-82cd-a96be9c08e4f.


## Timeline

- 2026-08-27T08:52:43Z @tobiu added the `bug` label
- 2026-08-27T08:52:44Z @tobiu added the `accessibility` label
- 2026-08-27T08:52:44Z @tobiu added the `agent-os` label
- 2026-08-27T08:52:44Z @tobiu added the `ai` label
- 2026-08-27T08:52:44Z @tobiu added the `design` label
- 2026-08-27T08:52:44Z @tobiu added the `regression` label
- 2026-08-27T08:52:45Z @tobiu added the `testing` label
- 2026-08-27T08:53:05Z @neo-gpt-emmy marked this issue as being blocked by #1
- 2026-08-27T08:54:39Z @tobiu cross-referenced by #3
- 2026-08-27T08:55:08Z @tobiu cross-referenced by #1
- 2026-08-27T09:04:00Z @tobiu cross-referenced by PR #5
- 2026-08-27T09:09:56Z @neo-gpt-emmy cross-referenced by PR #6
- 2026-08-27T11:12:44Z @neo-gpt cross-referenced by #25
- 2026-08-28T11:11:09Z @tobiu cross-referenced by PR #31
- 2026-08-28T11:19:23Z @neo-fable-clio cross-referenced by PR #32
- 2026-08-28T15:36:57Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T15:41:30Z @neo-fable-clio cross-referenced by PR #35
- 2026-08-28T19:58:16Z @tobiu referenced in commit `ceb707a` - "Merge pull request #35 from neomjs/agent/2-card-stale-validation

feat(agentos): the presence band projects provider validation provenance (#2)"
- 2026-08-28T19:58:16Z @tobiu closed this issue
- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T20:58:17Z @neo-fable-clio cross-referenced by PR #65

