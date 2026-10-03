---
id: 491
title: The state census fills its Accounts row from the config round-trip's four states
state: CLOSED
labels:
  - documentation
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T09:16:16Z'
updatedAt: '2026-10-03T10:44:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/491'
author: neo-opus-vega
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-03T10:44:54Z'
---
# The state census fills its Accounts row from the config round-trip's four states

## Context

`learn/CockpitStateCensus.md` merged as #489 (Resolves #478) at `59ae695` with its Accounts row marked `unknown` — the reason given: the cards' status text is composed at call time and had not been read. The read was done minutes later and pushed to the PR branch after the merge, so `dev` still carries the `unknown` row and candidate gap 9 ("Accounts · all cells unknown").

## The Problem

The one surface the census could not judge is the one the setup journey's Accounts cards present to a first-time operator; a walkthrough reader (#479) opening the page today finds a hole where the words should be.

## The Architectural Reality

The Accounts cards show no feed state. They render an ephemeral per-agent save status with four states — `pending · accepted · rejected · superseded` — produced by `apps/agentos/util/ConfigIntentRoundTrip.mjs` (`runConfigIntent`, `runPlaneCredentialIntent`) and sunk through `accounts/Panel.mjs` (`setAgentConfigSaveStatus` and its Repositories twin). Every rejection ends `Nothing was changed.`; only the installed-shell line names a next step.

## The Fix

Replace the `unknown` Accounts section with the six-column row sourced to those two symbols, and turn candidate gap 9 into the cell it is: `Accounts · rejected — next step`. Docs only; no code change. The commit exists (`a185080` on the #489 branch) and is re-based onto `dev`.

## Acceptance Criteria

- [ ] The Accounts section quotes the shipped sentences for the four states and the mode refusals, anchored to `ConfigIntentRoundTrip.mjs`'s two methods and `Panel.mjs`'s setters.
- [ ] Candidate gap 9 names the cell and the missing word (next step) instead of "all cells unknown".
- [ ] No other cell changes.

## Out of Scope

Any other census cell; the walkthrough (#479).

## Related

Parent: #477 (row 2). #478 (closed by #489, the census). The page is what #479 reads against.

Live latest-open sweep: the latest 20 open Institution issues were read at 2026-10-03T08:47Z and the board since; no leaf covers the Accounts row. A2A: no claim. Own-assignment: #485.

Origin Session ID: 075e6b2a-b93a-4972-b143-0fca9e7c06d8
Retrieval Hint: "state census Accounts row config round-trip pending accepted rejected superseded"

## Timeline

- 2026-10-03T09:16:17Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T09:16:18Z @neo-opus-vega added the `documentation` label
- 2026-10-03T09:16:18Z @neo-opus-vega added the `agent-os` label
- 2026-10-03T09:16:18Z @neo-opus-vega added the `ai` label
- 2026-10-03T09:16:35Z @neo-opus-vega added parent issue #477
- 2026-10-03T09:17:01Z @neo-opus-vega cross-referenced by PR #492
- 2026-10-03T09:53:03Z @neo-opus-vega cross-referenced by #495
- 2026-10-03T10:44:54Z @tobiu referenced in commit `8929b6d` - "docs(learn): the state census fills the Accounts row from the config round-trip's four states (#491) (#492)"
- 2026-10-03T10:44:54Z @tobiu closed this issue

