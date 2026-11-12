---
id: 359
title: Five outside readers follow the README's opening to a first action
state: OPEN
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-11-11T23:26:04Z'
updatedAt: '2026-11-11T23:26:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/359'
author: neo-fable-mnemosyne
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
# Five outside readers follow the README's opening to a first action

## Context

#356 rewrites the opening of `README.md` for a first-time reader. Its last criterion cannot be met by a pull request: readers outside the organization follow the line for their kind of reader from the page alone. #356 closes with its pull request, so this ticket holds that test and its result.

## The Problem

A cold read by a reviewer shows that the opening can be understood. It does not show that a person outside reaches a first useful action. Nobody has measured that for this package.

Three outside authors opened pull requests here on 2026-11-11, before the new opening existed (#351, #355, #357). Each was stopped by two mechanical checks, not by its change (#358). They are the baseline: people do arrive, and what stops them is recorded.

## The Architectural Reality

- After #356 the opening names four readers, each with one command (`README.md:1-23`).
- A first useful action is the reader's agent using one projected skill for real work: running a repository's unit tests through `unit-test`, opening a pull request through `pull-request`, or writing a view through `neo-app-authoring`.
- The app author's line depends on neomjs/create-app#59. Until `neo-app` 13.4.1 is on npm, a new workspace carries a skills release from before 1.0.0.

Design authority: "five readers outside the organization follow the line for their kind of reader from the page alone, and at least three reach a first useful action without help" (#356, Acceptance Criteria).

## The Fix

Ask five people outside the organization, with at least one app author and one contributor among them, to follow their line with nothing but the page. Record for each: which line, the command they ran, what their agent did first, and where they stopped.

## Acceptance Criteria

- [ ] Within a week of #356's merge five readers are asked. The ticket records when, and each reader by role; a name only where the reader agrees. Human step: finding the readers. Owner: @neo-fable-mnemosyne, with the operator's contacts where hers do not reach.
- [ ] The result is recorded here: how many answered, how many reached a first useful action, and where the others stopped.
- [ ] If fewer than three of five reach one, the opening goes back: a new ticket names the line that failed.

## Out of Scope

- Changing the opening before the result is in.
- Traffic, stars or download counts. Arrivals are upstream of the page.

## Related

#356 (the opening) · #358 (what stopped the three authors of 2026-11-11) · neomjs/create-app#59 (the app author's line waits on it)

Decision Record impact: none.

Ownership: self-assigned. Work starts when #356 merges.

Live latest-open + closed sweep: the latest 20 open by creation date (37 open in all, orange band; #356 owns the opening and closes with its pull request, no open ticket owns the reader test) and a search over all states (`reader test README outside readers`) at 2026-11-11T23:25Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the latest 30 messages of all read states at 23:08Z; no claim on this scope. MC sweep: the one run for #356 at 21:14Z ("an outside reader cannot tell what the package is or what to do first", 6 results, no prior decision). Own-assignment sweep: 2 open here (#356, #191); this is #356's post-merge criterion, moved. Structure map: N/A, no file is placed.

Origin Session ID: 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

Retrieval Hint: "five outside readers test of the skills README opening: first useful action, result record"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

## Timeline

- 2026-11-11T23:26:04Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-11-11T23:26:06Z @neo-fable-mnemosyne added the `documentation` label
- 2026-11-11T23:26:06Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-11T23:26:07Z @neo-fable-mnemosyne added the `contributor-experience` label
- 2026-11-11T23:26:07Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T23:27:11Z @neo-fable-mnemosyne cross-referenced by PR #360
- 2026-11-11T23:48:42Z @neo-fable-mnemosyne cross-referenced by #362
- 2026-11-11T23:48:51Z @neo-fable-mnemosyne cross-referenced by #363
- 2026-11-12T00:30:08Z @neo-fable-mnemosyne cross-referenced by #364
- 2026-11-12T00:44:49Z @neo-fable-mnemosyne cross-referenced by #356

