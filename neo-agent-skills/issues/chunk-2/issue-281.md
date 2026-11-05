---
id: 281
title: The shared PR baseline holds ten runner slots on every push
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - build
assignees: []
createdAt: '2026-11-05T15:24:08Z'
updatedAt: '2026-11-05T15:45:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/281'
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
---
# The shared PR baseline holds ten runner slots on every push

## Context

The neomjs organization runs on 20 concurrent job slots, and every repository's PRs share them. The operator, 2026-11-05: "we are limited to 20 runner slots, which makes smarter ci scoping crucial … it is intense with 20+ open PRs", and "ORG wide PRs as the limitation." The parent outcome is neomjs/neo#20328, cause 3 (job count).

## The Problem

`reusable-pr-baseline.yml` runs its ten checks as ten jobs: `version`, `release-ref`, `pr-base`, `skills-materialized`, `source-comment-archaeology`, `substrate-size`, `npm-overrides`, `secrets`, `commit-authorship` and `pr-body`. It does that on every push to every PR in every repository that calls it.

Each job holds a slot from boot to finish, while its own work takes 3 to 30 seconds. On neo PR #20332 (2026-11-05), three baseline jobs were still queued 25 minutes after the push, with 37 other checks already finished. The baseline is the fixed price of a push anywhere in the org, and at ten jobs it is the largest fixed block on an Engine push.

## The Architectural Reality

- Each of the ten jobs checks out the caller and installs the guards on its own.
- Callers name the job `baseline`, so the contexts read `baseline / <job name>`. neo's `dev` ruleset (19087298) requires all ten individually.
- Consumers pin a release tag (#116), so a consumer moves only when its pin moves.

Design authority: #15's AC reads "It exposes separately named base-branch and Skills-materialization jobs with stable names suitable for required status contexts." One job per check was chosen to give one context per check. This ticket challenges that choice under the slot cap, which #15 did not weigh.

## The Fix

Run the ten checks as steps of one `PR baseline` job, with one shared checkout and install. Each step runs `continue-on-error`, and a final step fails the job while naming every check that failed, so one red still shows all ten results.

Two ways to report, and this is the fork:
- **Recommended: one aggregate context**, `baseline / PR baseline`. Each consumer moves its pin and swaps its ten required contexts for the one, together. Ruleset edits are the operator's, so they go out in the same change.
- **Alternative: one job that posts ten commit statuses** under today's names. Rulesets stay untouched. The cost is `statuses: write`, which goes against #15's least-privilege contract, and fork PRs get read-only tokens and could never post them.

## Acceptance Criteria

- [ ] The workflow runs one job. Any failing check fails it, and the job summary names every failed check.
- [ ] `scripts/test-reusable-pr-baseline.mjs` pins the one-job shape, the per-check steps and the aggregate failure.
- [ ] A migration note per consumer: the pin bump and the ruleset context swap ship together, and the old pin keeps working until then.
- [ ] A consumer PR's baseline drops from ten jobs to one, measured on its check runs before and after.
- [ ] Patch version bump.

## Out of Scope

- What each check verifies.
- Engine-only workflows, which are the other leaves of neomjs/neo#20328.
- Required-context binding for Engine lints, which is neomjs/neo#17171.

## Avoided Traps

- **Dropping checks to save slots:** every check stays, only the job count changes.
- **A matrix:** that is still ten jobs.
- **Stopping at the first red:** an author would fix one check per push. The aggregate step reports all ten.

## Related

neomjs/neo#20328 (parent outcome) · #14 (PR governance across repositories) · #15 (the split this revisits) · #116 (tag pins) · #80 (baseline coordinates)

Sweeps at 2026-11-05T15:19Z found no equivalent:
- the latest open here and in neo;
- an all-state search on baseline / one job / runner / required contexts;
- A2A claims since 14:40Z;
- Memory Core on the problem nouns.

unowned-rationale: the fork changes rulesets in every consumer, so the operator's pick between the two report shapes comes first. I'll take it after that.
Origin Session ID: 8545cff9-f4d7-4731-b93e-7dfad1347c8a
Retrieval Hint: "PR baseline ten jobs one job runner slots org-wide 20 required contexts aggregate"

## Timeline

- 2026-11-05T15:24:09Z @neo-opus-grace added the `enhancement` label
- 2026-11-05T15:24:10Z @neo-opus-grace added the `ai` label
- 2026-11-05T15:24:10Z @neo-opus-grace added the `performance` label
- 2026-11-05T15:24:10Z @neo-opus-grace added the `build` label
- 2026-11-05T15:24:32Z @neo-opus-grace cross-referenced by #20338
- 2026-11-05T15:44:19Z @neo-opus-grace cross-referenced by #20340
### @neo-opus-grace - 2026-11-05T15:45:53Z

**Two findings for the one-job design** (2026-11-05, measured on neo)

1. **A body edit reruns all ten checks.** Callers fire on `edited` so that a corrected body goes green without a push, but only `pr-body` reads the body. The other nine judge an unchanged head again. Once the baseline is one job, an `edited` event can skip those nine steps and run `pr-body` alone.
2. **Superseded runs.** neo's caller had no concurrency group, so a superseded head's run waited in the queue and then ran anyway. neomjs/neo#20340 / PR #20341 fixes that in neo's caller:
   - a rerun-safe group;
   - `cancel-in-progress` off for `edited`, because `same-head-check-repair` reruns a cancelled run on the same head.

   Brain and Institution have their own `shared-pr-baseline.yml` callers, and those need the same three lines.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 8545cff9-f4d7-4731-b93e-7dfad1347c8a


- 2026-11-05T16:29:49Z @neo-gpt-emmy cross-referenced by #20347
- 2026-11-05T17:22:48Z @neo-opus-grace cross-referenced by PR #20349
- 2026-11-05T18:32:36Z @neo-opus-grace cross-referenced by PR #20362

