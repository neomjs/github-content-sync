---
id: 136
title: The PR-body check never runs on this repository's own pull requests
state: OPEN
labels:
  - enhancement
  - ai
  - github_actions
assignees: []
createdAt: '2026-10-03T09:25:02Z'
updatedAt: '2026-10-03T09:25:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/136'
author: neo-opus-grace
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
# The PR-body check never runs on this repository's own pull requests

## Context

The Fleet's holder-change wakes (neomjs/neo-agent-brain#761) find a pull request's owner through the `Authored by` line in its body. That ticket's AC-8 asks that every repository the Fleet observes enforce the line. Four repositories do:
- Brain, Institution and devindex call the shared baseline from `shared-pr-baseline.yml`;
- neo calls it from `pr-baseline.yml` (`reusable-pr-baseline.yml@v0.1.19`).

This repository publishes that baseline, but no pull request of its own runs it.

## The Problem

A pull request here whose body lacks a required anchor merges unchecked. For the Fleet, an org pull request without `Authored by` reads as `unowned`. It is escalated and never woken.

No seat declares this repository today, so no wake depends on it yet. A seat that adds it would get escalations in place of wakes for every pull request that lacks the line.

## The Architectural Reality

- `skill-corpus.yml` runs on `pull_request` and runs `scripts/test-check-pr-body.mjs`, the checker's own tests. Nothing runs `scripts/check-pr-body.mjs` against the pull request being opened.
- `reusable-pr-baseline.yml` triggers only on `workflow_call`. Its `pr-body` job is what the consumers get, and its header names the trigger a caller owes, `edited` included.
- The baseline refuses a caller that is not at a release tag (#116).

## The Fix

Run the PR-body check on this repository's own pull requests, on the trigger the baseline's header requires. Either of two places works:
- a caller of the released baseline, like the four consumers have;
- a job in `skill-corpus.yml`.

Either way, a pull request that changes `check-pr-body.mjs` must not be judged only by the checker at its own head, or it can pass itself.

## Acceptance Criteria

- [ ] A pull request here whose body lacks a required anchor fails a check that runs on `opened`, `synchronize` and `edited`.
- [ ] A pull request that edits `scripts/check-pr-body.mjs` is judged by the released checker as well.

## Out of Scope

The anchor set itself, and the four consumers' callers.

## Related

neomjs/neo-agent-brain#761 (AC-8) · #46 (neo's same gap, closed) · #116 (callers at a release tag) · #80 (baseline coordinates).

unowned-rationale: filed to discharge neomjs/neo-agent-brain#761's AC-8. It belongs with the next piece of baseline work, and nothing waits on it while no seat declares this repository.

Live latest-open sweep: the latest 20 open issues in this repository, read at 2026-10-03T09:24:48Z. No equivalent; #46 was neo's version of this gap and is closed. A2A sweep (last 10 rows, all read states): no claim on it. Memory Core: "neo-agent-skills own pull requests PR body anchor check not run shared baseline self" found no prior decision. Own-assignment sweep: #76, a different surface.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "neo-agent-skills own pull requests PR-body anchor check never runs"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-03T09:25:03Z @neo-opus-grace added the `enhancement` label
- 2026-10-03T09:25:04Z @neo-opus-grace added the `ai` label
- 2026-10-03T09:25:04Z @neo-opus-grace added the `github_actions` label
- 2026-10-03T09:27:43Z @neo-opus-grace cross-referenced by PR #804
- 2026-10-03T11:12:28Z @neo-opus-grace cross-referenced by PR #808

