---
id: 318
title: The shared PR baseline runs a caller's declared guards as one context
state: CLOSED
labels:
  - enhancement
  - ai
  - github_actions
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-11-08T06:41:15Z'
updatedAt: '2026-11-08T07:12:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/318'
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
closedAt: '2026-11-08T07:12:19Z'
---
# The shared PR baseline runs a caller's declared guards as one context

## Context

`neomjs/neo#17171`'s Decision (operator, 2026-11-08) retires a consumer's standalone guard workflows into **one aggregate required context, served by a shared step** in this repository's `reusable-pr-baseline.yml`. The rules stay in each repository and the mechanics are shared. This ticket is that Decision's sequence step 1: the shared step, then its release. The Engine half (its guard list, retiring its standalone workflows, the pin bump and the ruleset entry) is `neomjs/neo#17171` itself (@neo-opus-vega). Brain follows under its own ticket.

## The Problem

A consumer's repo-local guards run as standalone workflows. In Engine these are `guard-ci-parity-lint.yml`, `secrets-lint.yml` and `spec-retirement-lint.yml`. Each holds its own runner slot and context, and none is required, so each is advisory at merge. Requiring them one by one would cost a ruleset entry per guard, and a path-filtered required check that never runs blocks unrelated pull requests. The shared baseline reports the shared checks, but it has no place for a consumer's own.

## The Architectural Reality

- `.github/workflows/reusable-pr-baseline.yml` has two jobs:
  - `pr-baseline` runs every check that executes no pull-request code, as steps of one job (#281).
  - `skills-materialized` runs the caller's own install, and "never shares a job with the guards" (`:367-369`). Steps of one job share `GITHUB_ENV`, `GITHUB_PATH` and the filesystem, so pull-request code beside them could green every later step (`:8-11`).
- A repo-local guard executes the pull request's own scripts. That makes it pull-request code, so it cannot be a step of `pr-baseline`.
- GitHub Actions cannot generate steps from an input. A matrix over the list would cost one job (one runner slot, one context) per guard, the trap #281 already rejected.
- A skipped job cannot serve as a required context that a caller depends on (`:319-321`). Only a caller that declares guards requires `guards`, and for that caller the job always runs.
- The inputs are the consumer contract (README "## Versioning"). `scripts/test-reusable-pr-baseline.mjs` validates the workflow's shape and executes its inline scripts.

## The Fix

- **Input `guards`:** a string holding a JSON array of `{name, run}`, default `'[]'`.
- **Job `guards`** (`name: guards`), a sibling of `skills-materialized`, which runs only when the list is non-empty. It does checkout, `setup-node` (`inputs.node_version`), `npm ci`, then one step that runs each guard:
  - from the repository root, under `bash -e -o pipefail`, in order, each in its own log group;
  - every guard runs even after one fails;
  - the step summary gets a pass/fail table, and the job fails if any guard failed;
  - a malformed list fails the job before any guard runs: not JSON, not an array, an entry other than `{name, run}` with non-empty strings, a duplicate name, an unknown key.
- **The caller's context** is `<caller job> / guards`, which is `baseline / guards` in Engine.
- **README:** a "## Shared guard list" section with the input, the trust side, the context and a caller example. The header's job count changes.
- **`scripts/test-reusable-pr-baseline.mjs`:** arms for the job's shape and for its runner.
- **Version 1.1.0:** minor, because the input is opt-in and a caller that declares none sees no change.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `reusable-pr-baseline.yml` input `guards` | `neomjs/neo#17171` Decision | a JSON `[{name, run}]` list; each guard runs on the pull-request-code side | default `'[]'`: the job is skipped | README "## Shared guard list" | test arms |
| job and context `guards` | the same | one context per caller, red if any guard is red | none: skipped when the list is empty | the same | test arms |

## Acceptance Criteria

- [ ] The `guards` input defaults to `'[]'`. The `guards` job runs only for a non-empty list, beside `skills-materialized`, and shares no job with `pr-baseline`.
- [ ] Every declared guard runs, in order, even after a failure. The job is red if and only if a guard is red, and the step summary names each guard's result.
- [ ] A malformed list fails the job before any guard runs.
- [ ] `npm test` is green, with new arms for the shape and the runner.
- [ ] The version is 1.1.0, the README section exists, and the pull request names the bump and why.
- [ ] Post-merge: the `v1.1.0` release is published.

## Out of Scope

- Engine's guard list, retiring its standalone workflows, the pin bump and the ruleset entry (`neomjs/neo#17171`).
- Brain's adoption, under its own ticket.
- What any guard checks. The runner runs commands, and each guard stays in its repository.

## Avoided Traps

- **A step inside `pr-baseline`:** it mixes pull-request code into the job built to run none (see Reality).
- **A matrix:** a slot and a context per guard.
- **An always-running job:** a runner slot on every consumer's pull requests for nothing when none are declared.
- **Commit statuses per guard:** `statuses: write`, which a fork's read-only token cannot use (#281's reason).

## Related

`neomjs/neo#17171` (the Decision and the Engine half), #14, #281.

Live latest-open + closed sweep: checked the latest 20 open + closed searches ("guards", "guard list", "aggregate context", "repo-local lint", "reusable-pr-baseline input"; #281's Out of Scope and Avoided Traps read) at 2026-11-08T06:40Z; no equivalent or prior exclusion found. A2A in-flight sweep (last 30, all read states): no other claim on this scope. Memory Core sweep (the problem's nouns): no prior attempt. Own-assignment sweep: no overlap.

unowned-rationale: none — self-assigned at creation; 13.4 waits on it.

Origin Session ID: d2ffd5a3-0c0e-4f7a-b635-04e860cc8d4f
Retrieval Hint: "shared PR baseline guards input aggregate context repo-local guard workflows"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-11-08T06:41:15Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-08T06:41:16Z @neo-opus-grace added the `enhancement` label
- 2026-11-08T06:41:16Z @neo-opus-grace added the `ai` label
- 2026-11-08T06:41:17Z @neo-opus-grace added the `github_actions` label
- 2026-11-08T06:41:24Z @github-actions added the `needs-design-authority` label
- 2026-11-08T06:50:33Z @neo-opus-grace cross-referenced by PR #319
- 2026-11-08T07:12:19Z @tobiu referenced in commit `6a728b3` - "feat(baseline): the shared PR baseline runs a caller's declared guards as one context (#318) (#319)

A guards input takes a JSON list of {name, run}. A guards job beside Skills materialized, on the pull-request-code side, runs every guard after npm ci, even after a red, and fails if any did, so a caller binds one context however many guards it declares. A malformed list fails before any guard runs, and a caller that declares none skips the job. The contract test runs the extracted runner for real, and anchors its job lookup to a line start, since the guards input's line contains the job's marker. 1.1.0: a minor bump, because the input is opt-in."
- 2026-11-08T07:12:19Z @tobiu closed this issue

