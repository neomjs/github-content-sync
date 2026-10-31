---
id: 845
title: 'A revoked PAT can take ~10 min to reach the spine: two 300 s intervals stack'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-30T22:44:54Z'
updatedAt: '2026-10-31T19:06:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/845'
author: neo-opus-ada
commentsCount: 0
parentIssue: 424
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T19:06:32Z'
---
# A revoked PAT can take ~10 min to reach the spine: two 300 s intervals stack

## Context

In row 5's H3 on 2026-10-30 ([#516 receipt 6534998715](https://github.com/neomjs/neo-agent-institution/issues/516#issuecomment-6534998715)), a fresh PAT was revoked at 22:17:19Z. "pat refused" showed in the spine between 22:20:49Z and 22:21:19Z, under 4 minutes; the alignment happened to be favourable. The step list (#516, 6457088113) promises the refusal **within 5 minutes**.

## The Problem

Two independent 300 s intervals add up:
- the cockpit re-checks the launched PAT every `planeCredentialRecheckInterval` = 300 000 ms (`apps/agentos/view/fleet/cockpit/Container.mjs`, via `PlaneCredentialCheck`);
- the plane caches a PAT verification for `NEO_AUTH_PAT_CACHE_TTL_SECONDS` = 300 s.

A re-check that lands just before the cache expires sees a stale admission, and the next one comes 300 s later. The worst case is therefore close to 600 s, double the documented bound.

## The Fix

Make the worst case fit the bound. Two options:
- re-check more often than the cache TTL, for example every 60 s with the existing cheap verify;
- or have the plane drop its cached admission when any request's GitHub check returns 401, with the cockpit treating a 401-class refusal on any plane read as immediate "pat refused".

Alternatively, if 10 min is accepted, update the documented bound in #424/#516. That is the operator's call.

## Acceptance Criteria

- [ ] AC-1 A spec with a fake clock shows a revoke reaching "pat refused" within the documented bound in the worst-case alignment.
- [ ] AC-2 The bound written in #424/#516 matches the implementation.

## Related

#424, #516, #828 / PR #832 (the refusal reaches the spine), #843 (no restart on PAT update).

Live latest-open + closed sweep: checked the latest 20 open issues plus a search ("PAT recheck interval cache") at 2026-10-30T22:5xZ; no equivalent. A2A: no claim on this scope.

unowned-rationale: Ready for an FM maintainer; Ada stewards row 5 and takes it if unclaimed.

Retrieval Hint: "PAT revoke detection bound planeCredentialRecheckInterval cache TTL stack"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f

## Timeline

- 2026-10-30T22:44:56Z @neo-opus-ada added the `bug` label
- 2026-10-30T22:44:56Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T22:44:56Z @neo-opus-ada added the `ai` label
- 2026-10-30T22:45:41Z @neo-opus-ada added parent issue #424
- 2026-10-31T17:56:28Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-31T18:21:44Z @neo-gemini-maryam cross-referenced by PR #851
- 2026-10-31T18:26:55Z @neo-gemini-maryam referenced in commit `f35989d` - "fix(agentos): stamp visual baseline inputs for #845 (#845)"
- 2026-10-31T18:45:43Z @neo-gemini-maryam referenced in commit `2ed0b13` - "fix(agentos): set DEFAULT_RECHECK to 45 s for 360 s off-grid bound (#845)"
- 2026-10-31T18:54:16Z @neo-opus-ada cross-referenced by #424
- 2026-10-31T18:54:38Z @neo-opus-ada cross-referenced by #516
- 2026-10-31T19:06:32Z @tobiu referenced in commit `c0da3d0` - "fix(agentos): bound revoked-PAT spine detection to 6 min via 45 s re-check (#845) (#851)

* fix(agentos): bound revoked-PAT spine detection to 6 min via 60 s re-check (#845)

* fix(agentos): stamp visual baseline inputs for #845 (#845)

* fix(agentos): set DEFAULT_RECHECK to 45 s for 360 s off-grid bound (#845)"
- 2026-10-31T19:06:33Z @tobiu closed this issue

