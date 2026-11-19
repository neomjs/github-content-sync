---
id: 1857
title: 'A refused Antigravity binding logs its seat, refusal code and generation'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-19T13:23:22Z'
updatedAt: '2026-11-19T14:17:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1857'
author: neo-opus-grace
commentsCount: 0
parentIssue: 1531
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1856 Antigravity 2.33.0 passes admission but fails every seat''s binding'
closedAt: '2026-11-19T14:17:05Z'
---
# A refused Antigravity binding logs its seat, refusal code and generation

## Context

On 2026-11-19 Antigravity 2.33.0 refused the project binding on every Gemini seat Start (#1856). The Fleet stopped each seat, and nothing recorded which binding step refused. `FleetLifecycleService#finishAntigravityStart` throws `ANTIGRAVITY_BINDING_FAILED` with the refusal as `cause`. `FleetControlBridge#startOutcome` turns that throw into a rejection for the cockpit (`START_REFUSAL_CALLERS`) and logs nothing. `~/Library/Logs/Neo Harness/main.log` held no line for any of the four Starts.

Split from #1856, whose fallback design needs this line in the installed Fleet first: one controlled 2.33.0 Start then names the refusal.

## The Problem

Without the refusal code nobody can tell a slow first launch (`listener-unproven`, `renderer-unproven`) from a deterministic break (`route-mismatch`, `project-registry-unproven`), or which build caused it. The operator only sees "the seat was stopped before admission activation".

## The Architectural Reality

- `ai/services/fleet/FleetLifecycleService.mjs#finishAntigravityStart`: `failureCause` holds the binding callback's error. An `AntigravityProjectBindingRefusal` carries a stable `code` that never contains profile or project contents. `failureReason` holds a launch-admission reason.
- The Start's `opts.remoteMcpCapability.harnessGeneration` names the admitted build (`generationId`, `shortVersion`).
- The Fleet log's precedent is `failureWords`: the cockpit gets fixed words, and the Fleet log keeps the cause.
- Design authority: n/a. This adds a log line and calls no behaviour wrong.

## The Fix

In `finishAntigravityStart`, on any failure that is not a Stop, log one line:
`[FleetLifecycleService] seat '<id>' Antigravity binding refused (<code>) on generation <shortVersion> (<generationId, 12 chars>); the seat was stopped.`
The code is the cause's own code when it is a short token, else the launch-admission reason. When cleanup is unresolved, the line ends "its cleanup is unresolved." A Stop during binding logs nothing.

## Acceptance Criteria

- [ ] A refused binding logs one line naming the seat, the refusal code, and the generation's version and id.
- [ ] A Stop during binding logs nothing.
- [ ] No callback error text reaches the line; the existing "does not expose callback error text" test still passes.

## Out of Scope

The first-Start proof, the fallback retry and rejection (#1856).

## Related

Parent #1531 · #1856

Live latest-open + closed sweep: checked the latest open issues and searched "binding refusal log" (open and closed) at 2026-11-19T13:23Z; #1856 is the only overlap, and this ticket is split from it. Open count 161 (red band): a leaf of #1531. A2A: my own `[lane-claim]` of #1856 at 13:19Z covers it.

Origin Session ID: 5a375982-a381-43b6-a753-4fb8e933059d

Retrieval Hint: "Antigravity binding refused log line, finishAntigravityStart failureCause code, generation id"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-19T13:23:22Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-19T13:23:23Z @neo-opus-grace added the `bug` label
- 2026-11-19T13:23:23Z @neo-opus-grace added the `ai` label
- 2026-11-19T13:23:24Z @neo-opus-grace added the `agent-os` label
- 2026-11-19T13:23:31Z @neo-opus-grace added parent issue #1531
- 2026-11-19T13:23:34Z @neo-opus-grace marked this issue as blocking #1856
- 2026-11-19T13:24:00Z @neo-opus-grace cross-referenced by #1856
- 2026-11-19T13:25:08Z @neo-opus-grace cross-referenced by PR #1858
- 2026-11-19T13:44:30Z @neo-opus-grace cross-referenced by #1859
- 2026-11-19T14:17:04Z @tobiu referenced in commit `c8ce639` - "fix(fleet): a refused Antigravity binding logs its seat, refusal code and generation (#1857) (#1858)

On 2026-11-19 Antigravity 2.33.0 refused every Gemini seat's project binding, and no log named the refused step: the Start answers the cockpit with a fixed refusal. finishAntigravityStart now logs one line per refused binding (never for a Stop) with the cause's code, or the launch-admission reason, and the harness generation. Split from #1856, whose controlled 2.33.0 Start reads this line."
- 2026-11-19T14:17:05Z @tobiu closed this issue
- 2026-11-19T14:32:47Z @neo-gemini-julia cross-referenced by PR #1860

