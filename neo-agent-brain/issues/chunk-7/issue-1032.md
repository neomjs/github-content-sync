---
id: 1032
title: Antigravity endpoint wait's deadline refusal depends on timer order
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-14T19:59:52Z'
updatedAt: '2026-10-15T09:45:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1032'
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
closedAt: '2026-10-15T09:45:30Z'
---
# Antigravity endpoint wait's deadline refusal depends on timer order

## Context
Brain #1031's body reports that the unchanged spec test `keeps a missing profile endpoint within the bounded Start timeout` once returned `timeout` instead of `stale-endpoint`, then passed in isolation. Reproduced independently at #1031's head `1dffa89b`, where both the test and the loop are unchanged from `dev` `dfd9a3e7`: 60 of 60 passed at `--repeat-each=60` on 9 workers; 1 of 400 failed at `--repeat-each=400 --workers=24` (`Expected "stale-endpoint"`, `Received "timeout"`).

## The Problem
When a profile endpoint never becomes valid, which refusal the binding reports depends on which of two timers set for the same deadline fires first. The spec flakes. Brain's unit gate compares head failures with base failures, so a failure that occurs only on head turns an unrelated PR red, as the 10-14 `receiver.spec` sighting on #1027 showed.

## The Architectural Reality
`waitForOwnedProfileEndpoint` in `ai/services/fleet/antigravityProjectBinding.mjs` polls until `deadline` and records the last observation in `lastRefusal`. Once `remaining(deadline) > 0` fails, it throws `refuse(lastRefusal)`. Between polls it sleeps through `within(new Promise(resolve => setTimeout(resolve, Math.min(POLL_INTERVAL, remaining(deadline)))), deadline, signal)`. On the last poll, that sleep and `boundedWait`'s own deadline timer target the same instant. If `Date.now()` advances between the two `remaining()` reads, the deadline timer is 1 ms shorter. It then rejects with `timeout` before the loop can report `lastRefusal`.

Design authority: the loop's `lastRefusal` contract and its spec. `keeps a missing profile endpoint within the bounded Start timeout` asserts `{code:'stale-endpoint'}`. The spec's only `timeout` expectation belongs to the registry reader (`closes a registry descriptor that opens after timeout`).

## The Fix
In `waitForOwnedProfileEndpoint`, a deadline reached while polling reports `lastRefusal`: the inter-poll sleep no longer races the deadline with its own `timeout`, and it still rejects `canceled` on abort. If a poll step is still running when the deadline passes, choose `timeout` or `lastRefusal` once and state the choice in the JSDoc. Pin the deadline-on-sleep case with a deterministic control (a clock or sleep seam) instead of repeat counts.

## Acceptance Criteria
- [ ] An endpoint that never becomes valid refuses with the last observed refusal, whatever the timer order.
- [ ] A deterministic unit test makes the deadline land on the final inter-poll sleep.
- [ ] Abort during the sleep still refuses `canceled`.
- [ ] PR receipt: the existing test at `--repeat-each=400 --workers=24` shows 0 failures.

## Out of Scope
Other timing-dependent Brain specs (the wake receiver's stuck-step naming is a separate sighting); the unit gate's rerun behavior; binder steps after the endpoint proof.

## Avoided Traps
Accepting either code in the assertion, which hides the production nondeterminism. Raising the 180 ms budget, which narrows the window but keeps the race.

Related: #1017 (where the binder originated); #1031 (where this surfaced).

handoff: @neo-gpt-emmy (binder author; proposed, not assigned)

Live latest-open sweep: latest 20 open Brain issues at 2026-10-14T19:58:59Z, no equivalent. A2A sweep: latest 30 messages in all states, no competing claim. MC sweep: one problem-noun query, 6 results, no prior decision. Own-assignment sweep: 15 open; #571 is the only same-surface parent, and its body parks no binder timing work. Exact sweep: `stale-endpoint` and `waitForOwnedProfileEndpoint` across neomjs, no ticket. Structure map: existing owner `ai/services/fleet/antigravityProjectBinding.mjs`, no new module.
Decision Record impact: none.

Origin Session ID: c51bd013-6b01-4e9d-af21-8f73cfc74008
Retrieval Hint: "Antigravity profile endpoint deadline refusal timeout stale-endpoint timer order"

Authored by Ada (@neo-opus-ada), Claude Opus 5.5, Claude Code.

## Timeline

- 2026-10-14T19:59:53Z @neo-opus-ada added the `bug` label
- 2026-10-14T19:59:54Z @neo-opus-ada added the `ai` label
- 2026-10-14T19:59:54Z @neo-opus-ada added the `testing` label
- 2026-10-14T19:59:54Z @neo-opus-ada added the `agent-os` label
- 2026-10-14T20:01:51Z @neo-opus-ada cross-referenced by PR #1031
- 2026-10-15T09:09:17Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-15T09:23:02Z @neo-gemini-julia cross-referenced by PR #1039
- 2026-10-15T09:38:01Z @neo-opus-ada cross-referenced by #1042
- 2026-10-15T09:45:30Z @tobiu referenced in commit `21b9ee9` - "fix(fleet): report last endpoint refusal when deadline expires (#1032) (#1039)

Co-authored-by: Ada <neo-opus-4-7@neomjs.com>"
- 2026-10-15T09:45:30Z @tobiu closed this issue
- 2026-10-15T10:15:45Z @neo-gemini-julia cross-referenced by PR #1045

