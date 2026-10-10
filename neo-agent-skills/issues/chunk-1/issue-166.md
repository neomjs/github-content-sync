---
id: 166
title: The core-idiom trigger names test fixtures and test apps as idiom references
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-10T22:29:33Z'
updatedAt: '2026-10-10T22:31:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/166'
author: neo-opus-ada
commentsCount: 1
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
# The core-idiom trigger names test fixtures and test apps as idiom references

## Context

The neo #19574 PR added a component test app (`test/playwright/component/apps/grid-component-columns/app.mjs`). Its toggle misused three engine idioms:
- `store.setData(..., {continuation: true})` was used as a scroll-keeping switch on a whole-corpus replacement;
- records were rebuilt from `toJSON()` to change one field, instead of `record.set()`;
- the grid was resolved by a hard-coded `Neo.get('<id>')` from a sibling button.

The review approved it. The operator caught all three, and the reviewer placed a merge hold (#19574, 22:20Z). A test app is what the next seat copies.

## The Problem

`references/pr-review-guide.md` §7.5.1 loads the core-idiom audit for diffs that create, mutate, resolve or destroy Neo instances "(any dir)". Test fixtures and test apps qualify, but a reviewer reads them as plumbing and does not apply the audit there.

## The Fix

Add one clause to §7.5.1's trigger: test fixtures and test apps are idiom references the next seat copies, so they are held to the same checks. Bump the package version from 0.1.38 to 0.1.39.

## Acceptance Criteria

- [ ] §7.5.1 names test fixtures and test apps as in scope, in one clause.
- [ ] `lint-skill-corpus` passes, and the package ships as 0.1.39.
- [ ] Post-merge: the next review of a PR that adds or changes a test app applies the audit's checks to it, as observed in a review body.

## Out of Scope

- The audit's checks themselves.
- Lint enforcement. Sunset: this clause retires when an idiom lint covers test code.

## Related

neo #19574 (the evidence), #155 / #163 (the audit's last change), #61 (triggers that do not fire).

Sweeps:
- Live latest-open sweep: the latest 20 open issues at 22:29:14Z; no equivalent. #61 covers triggers not firing in general.
- A2A in-flight sweep: Vega proposed this to Ada and files nothing.
- Memory sweep: no prior ruling.
- Own-assignment sweep: #102, unrelated.

Origin Session ID: a6673f82-f995-4e40-83a9-3093d4ecd357
Retrieval Hint: "core-idiom audit trigger test fixtures test apps idiom references plumbing"


## Timeline

### @neo-opus-ada - 2026-10-10T22:29:48Z

Closed: filed incomplete by an interrupted call. The proposal is parked for now.

- 2026-10-10T22:29:49Z @neo-opus-ada closed this issue
- 2026-10-10T22:31:11Z @neo-opus-ada reopened this issue
- 2026-10-10T22:31:12Z @neo-opus-ada added the `enhancement` label
- 2026-10-10T22:31:12Z @neo-opus-ada added the `ai` label
- 2026-10-10T22:31:13Z @neo-opus-ada assigned to @neo-opus-ada

