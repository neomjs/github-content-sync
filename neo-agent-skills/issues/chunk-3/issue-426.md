---
id: 426
title: 'A PR can add CI jobs without a word, though the org has 20 runner slots'
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - build
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-19T17:45:34Z'
updatedAt: '2026-11-19T19:50:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/426'
author: neo-opus-grace
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
closedAt: '2026-11-19T19:50:42Z'
---
# A PR can add CI jobs without a word, though the org has 20 runner slots

## Context

The operator, 19 Nov 2026: CI has 20 concurrent runner jobs across the org, and the team aggregated its jobs to fit (neomjs/neo#20328 in Engine, #281 and #318 here). Since then the count has grown back, and nothing told anyone. His words: "peers should get a warning whenever they add more into CI. it must be clear that we have 20 runner jobs, and peers need to justify why they do not aggregate."

Measured on 19 Nov (Engine PR heads `760009ebe3`, `e156a55121` and the `dev` head):

| Event | Runner jobs |
|---|---|
| A full Engine PR push | about 35 |
| A `dev` push | 26 |
| A PR body edit | 7, merged PRs included |

Added in Engine since #20328 closed on 8 Nov:
- engine e2e shards 14 → 15 (neomjs/neo#21288)
- the `red-on-base` advisory job (neomjs/neo#21027)
- the claim responder on every PR event

Each was a reasonable call on its own, and no step asked about the budget.

## The Problem

A change that starts more runner jobs looks like any other diff. The PR body has no slot for the cost, no check counts it, and no review audit asks about it. The budget lives in one operator's head and in closed tickets, so every addition is judged without it.

## The Architectural Reality

- `.github/workflows/reusable-pr-baseline.yml`: its `pr-baseline` job fetches the live body (`BODY_FILE`) and runs `neo-agent-skills-pr-body`, for every consumer repository.
- `scripts/check-pr-body.mjs`: the body rules (anchors, close target, image embeds, frame critique in #425).
- `scripts/check-workflow-concurrency.mjs`: precedent for reading `.github/workflows/*.yml` line by line, with no YAML dependency. The package depends only on `acorn` and `semver`.
- `pull-request-workflow.md` §9: the PR-body template.

## The Fix

1. **`scripts/check-ci-jobs.mjs`** (bin `neo-agent-skills-ci-jobs`) estimates, at base and at head, the runner jobs that a `pull_request` or `push` event can start: each job once, times its matrix (literal lists, or the fallback list of a `fromJSON(… || '[…]')`). The estimate only has to be consistent, since the check compares base and head. When head starts more, the body must carry `CI jobs: +N — <why it cannot join an existing job>`. Without that line the check fails with the budget in its message: 20 concurrent runner jobs across the org, about 35 already on a full Engine PR push. The job and matrix entries that grew are named. *(Shipped in #428 as `scripts/ci-job-count.mjs` behind `neo-agent-skills-pr-body --base <ref>`, with no separate bin.)*
2. **The baseline runs it** in the `pr-baseline` job, which already reads the body, so every consumer gets it with the next pin.
3. **The template** gains the slot: a PR that adds CI jobs carries a `CI jobs:` line.
4. **Review:** the line's reason is a Required Action item when the added work could join an existing job.
5. The version is bumped.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-skills-pr-body --base <ref>` (`scripts/check-pr-body.mjs`), counting with `scripts/ci-job-count.mjs` | the operator, 19 Nov | fails a PR whose workflows start more jobs than the commit it forked from (`git merge-base <ref> HEAD`), unless the body carries a filled `CI jobs: +N — <reason>` line | no workflow change or fewer jobs: passes; a tree without `.github/workflows`: no jobs; any other read failure, or a base with no shared history: fails | the scripts' JSDoc | contract arms in `scripts/test-check-pr-body.mjs` |
| PR template slot | the same | `CI jobs: +N — <reason>` | absent when no job is added | `pull-request-workflow.md` §9 | AC-3 |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1: the check counts jobs and literal or fallback matrix entries at base and head. A new workflow, a new job or a longer matrix list each raise the count. Contract arms cover each one, plus a fall and no change.
- [ ] AC-2: a rise without a `CI jobs:` content line fails, and the message names the 20-slot budget and the entries that grew. The line in a fence or inline code does not count.
- [ ] AC-3: the shared baseline runs it on agent PRs, and the template carries the slot.
- [ ] AC-4: the version is bumped.

## Post-Merge Validation

- [ ] After Engine's next pin bump, a PR that adds a shard or a job is asked for the line.

## Out of Scope

- Removing today's jobs. #427 cuts the shared baseline's own extra runs, and an Engine ticket cuts Engine's.
- Counting jobs that run on `schedule` or `workflow_dispatch` only.

## Avoided Traps

- **A YAML dependency.** The comparison needs consistency, not a full parse, and the package keeps its two dependencies.
- **A hard cap.** A justified addition stays possible. The rule asks for the reason, which is what the operator asked for.

## Related

#427 · #281 · #318 · #14 · neomjs/neo#20328 · neomjs/neo#21494 (the sixteenth shard, declined on this budget)

Live sweep at 2026-11-19T17:37Z: I read the latest 20 open issues here and in Engine, and searched open and closed issues for runner jobs, runner slots, CI jobs, same-head, aggregate workflow and red-on-base. Open owners: none. #281 and neomjs/neo#20328 are closed and kept their own Out of Scope. Backlog: 39 open (orange). No owning open epic: #14 is about where governance runs, not this budget. A2A: no claim on CI budget in the last 30 messages. Memory Core: only today's measurement. My own open assignments: none on this surface.

Ownership: self-assigned. I implement it after #425 lands, since both touch the PR-body check.

Origin Session ID: 41be8b70-720d-4106-828c-036c564b0fb7
Retrieval Hint: "CI runner budget 20 slots, a PR that adds jobs must justify not aggregating"


## Timeline

- 2026-11-19T17:45:35Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-19T17:45:36Z @neo-opus-grace added the `enhancement` label
- 2026-11-19T17:45:37Z @neo-opus-grace added the `ai` label
- 2026-11-19T17:45:37Z @neo-opus-grace added the `performance` label
- 2026-11-19T17:45:37Z @neo-opus-grace added the `build` label
- 2026-11-19T17:45:55Z @neo-opus-grace cross-referenced by #427
- 2026-11-19T17:46:04Z @github-actions added the `needs-design-authority` label
- 2026-11-19T18:12:09Z @neo-opus-grace cross-referenced by PR #428
- 2026-11-19T18:16:47Z @neo-opus-grace referenced in commit `7ea5aab` - "fix(pull-request): the job counter strips comments in linear time (#426)

CodeQL flagged `(^|\s+)#.*$` in ci-job-count.mjs as a polynomial regular
expression on library input: a long run of spaces made each start position
retry. A scan for the first `#` that opens the value or follows whitespace
replaces both uses. The key-line match drops its `$` anchor, which a CRLF
line could make backtrack, and the indent is read without a regular
expression.

A 50,000-space adversarial line now takes 0 ms. Engine's dev still counts
38 runner jobs, and a CRLF workflow counts correctly."
### @neo-opus-grace - 2026-11-19T18:29:14Z

**Session handover (19 Nov, 18:25Z) — owner: @neo-opus-grace, continuing in a fresh session.**

- **State:** PR #428 at `7ea5aab` is green (CodeQL included) and requested from @neo-gemini-julia for a full review. The branch is `grace/426-ci-jobs-justified`; the worktree is `neo-agent-skills-426`.
- **Shape:**
  - `scripts/ci-job-count.mjs` counts runner jobs per workflow, a job once per matrix entry, line-based. It counts 38 on Engine's `dev`.
  - `neo-agent-skills-pr-body --base <ref>` refuses more jobs without a `CI jobs:` line.
  - The shared baseline's anchors step passes `--base`, with 2 contract mutations.
  - The sparse head checkout in `skill-corpus.yml` carries the counter.
  - Version 2.2.0.
  - Receipt: on Engine's sixteenth-shard commit it reports "38 → 39 (test-e2e.yml#e2e-engine 15 → 16)".
- **Pickup:** answer Julia's review; I flagged the seam choice (the rule inside the one agent-gated step) for her to challenge. After merge and Engine's pin bump, run the Post-Merge check: an agent PR adding a job is refused without the line.
- **Next:** #427 touches the same workflow, so it starts after this merges.

- 2026-11-19T19:20:40Z @neo-opus-grace referenced in commit `415458c` - "fix(pull-request): the CI jobs check compares from the fork point and reads from the repository root (#426)

A job the base cut after the branch forked is no longer charged to the branch, both sides read from the repository root whatever the working directory, and only an absent .github/workflows reads as no jobs: any other read failure throws instead of counting zero. A CI jobs line needs a number and a filled reason."
- 2026-11-19T19:50:42Z @tobiu referenced in commit `660a263` - "feat(pull-request): a diff that starts more CI runner jobs says why it cannot aggregate (#426) (#428)

* feat(pull-request): a diff that starts more CI runner jobs says why it cannot aggregate (#426)

The operator, 19 Nov: CI has 20 concurrent runner jobs across the org, the
team aggregated to fit, and "peers need to justify why they do not
aggregate". A full Engine PR push already starts about 35.

- scripts/ci-job-count.mjs estimates the runner jobs a repository's
  workflows start on pull_request, pull_request_target or push. It counts
  each job times its matrix: a literal list, a block list or a fromJSON
  fallback list. Line-based, no YAML dependency. On Engine's dev it counts 38.
- check-pr-body.mjs --base <ref> compares the base's workflows (from git)
  with the checked-out tree. More jobs without a `CI jobs:` content line is
  a visible finding naming the budget and each grown file#job.
- The shared baseline's anchors step passes --base from the workspace.
  Two contract mutations pin both.
- skill-corpus.yml's sparse head checkout carries the counter too, since
  the guard imports it.
- The PR template gains the `CI jobs:` slot. 2.2.0.

On Engine's sixteenth-shard commit 0255c9a77c against its parent, the check
reports "38 → 39 (test-e2e.yml#e2e-engine 15 → 16)".

* fix(pull-request): the job counter strips comments in linear time (#426)

CodeQL flagged `(^|\s+)#.*$` in ci-job-count.mjs as a polynomial regular
expression on library input: a long run of spaces made each start position
retry. A scan for the first `#` that opens the value or follows whitespace
replaces both uses. The key-line match drops its `$` anchor, which a CRLF
line could make backtrack, and the indent is read without a regular
expression.

A 50,000-space adversarial line now takes 0 ms. Engine's dev still counts
38 runner jobs, and a CRLF workflow counts correctly.

* fix(pull-request): the CI jobs check compares from the fork point and reads from the repository root (#426)

A job the base cut after the branch forked is no longer charged to the branch, both sides read from the repository root whatever the working directory, and only an absent .github/workflows reads as no jobs: any other read failure throws instead of counting zero. A CI jobs line needs a number and a filled reason."
- 2026-11-19T19:50:44Z @tobiu closed this issue
- 2026-11-19T20:45:26Z @neo-gemini-julia cross-referenced by PR #429

