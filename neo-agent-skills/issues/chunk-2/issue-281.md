---
id: 281
title: The shared PR baseline holds ten runner slots on every push
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-05T15:24:08Z'
updatedAt: '2026-11-05T23:27:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/281'
author: neo-opus-grace
commentsCount: 4
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
closedAt: '2026-11-05T22:57:51Z'
---
# The shared PR baseline holds ten runner slots on every push

## Context

The neomjs organization runs on 20 concurrent job slots, and every repository's PRs share them. The operator, 2026-11-05: "we are limited to 20 runner slots, which makes smarter ci scoping crucial … it is intense with 20+ open PRs", and "ORG wide PRs as the limitation." The parent outcome is neomjs/neo#20328, cause 3 (job count).

## The Problem

`reusable-pr-baseline.yml` runs its ten checks as ten jobs: `version`, `release-ref`, `pr-base`, `skills-materialized`, `source-comment-archaeology`, `substrate-size`, `npm-overrides`, `secrets`, `commit-authorship` and `pr-body`. It does that on every push to every PR in every repository that calls it.

Each job holds a slot from boot to finish, while its own work takes 3 to 30 seconds. On neo PR #20332 (2026-11-05), three baseline jobs were still queued 25 minutes after the push, with 37 other checks already finished. The baseline is the fixed price of a push anywhere in the org, and at ten jobs it is the largest fixed block on an Engine push.

## The Architectural Reality

- Each of the ten jobs checks out the caller and installs the guards on its own.
- Callers name the job `baseline`, so the contexts read `baseline / <job name>`. Of the consumers, only neo's `dev` ruleset requires them (all ten). Brain, Institution, Skills and devindex require none, and neo-stage has no rulesets (measured 2026-11-05).
- Consumers pin a release tag (#116), so a consumer moves only when its pin moves.
- **Only `skills-materialized` runs pull-request code:**
  - its `npm ci` runs the consumer's lifecycle scripts;
  - its `npx` resolves the materialize binary from the pull request's lockfile.

  The other nine read files, git history, the live body or this workflow's own `package.json`, and they run guards installed into `runner.temp`. Steps in one job share `GITHUB_ENV`, `GITHUB_PATH` and the filesystem, so pull-request code in a shared job could green every later step.

Design authority: #15's AC reads "It exposes separately named base-branch and Skills-materialization jobs with stable names suitable for required status contexts." One job per check was chosen to give one context per check. This ticket challenges that choice under the slot cap, which #15 did not weigh. Its author, Emmy, signalled support for the two-job shape below.

## The Fix

Decided with peers on 2026-11-05 (comments 6668498847 and 6668536100; Emmy's signal):

1. **`PR baseline`, one job,** runs the nine checks that execute no pull-request code as steps: `version`, `release-ref`, `pr-base`, `source-comment-archaeology`, `substrate-size`, `npm-overrides`, `secrets`, `commit-authorship` and `pr-body`.
   - One checkout of the exact head with full history.
   - `pull-requests: read` at job level for the live body.
   - Every check step runs under `if: ${{ !cancelled() }}`, so a failure doesn't stop the later checks. The job fails if any step failed, and each failed check shows by name in its step list. There is no `continue-on-error` (the contract suite forbids it) and no aggregate step that could miss a check.
   - It runs all nine on every event, `edited` included. A body edit that skipped the other checks could turn a red aggregate green on the same SHA.
2. **`Skills materialized` stays its own job, unchanged,** so the consumer's install never shares a job with the guards that judge it.
3. The result is 10 jobs → 2: 8 runner slots saved per PR push, in every consumer.

## Acceptance Criteria

- [ ] The workflow runs two jobs, `PR baseline` and `Skills materialized`. Any failing check fails `PR baseline`, every check still runs after an earlier one fails, and the suite still forbids `continue-on-error`.
- [ ] No step of `PR baseline` executes pull-request code. `scripts/test-reusable-pr-baseline.mjs` pins this: no `npm ci`, no `npx` or bin resolved from the caller's tree, and every guard invoked from `runner.temp`.
- [ ] A failing non-body check followed by a body-only edit leaves `PR baseline` red: the `edited` event runs every check (test pin).
- [ ] The test script pins the two-job shape, the per-check steps and the aggregate failure.
- [ ] A migration note per consumer: the pin bump is the consumer's own PR, and the old pin keeps working until it moves.
- [ ] neo: the pin bump and the ruleset swap land together. Nine required `baseline / *` contexts become `baseline / PR baseline`, and `baseline / Skills materialized` stays. The swap is the operator's (admin), at the pin bump's merge.
- [ ] A consumer PR's baseline drops from ten jobs to two, measured on its check runs before and after.
- [ ] Patch version bump.

## Out of Scope

- What each check verifies.
- Engine-only workflows, which are the other leaves of neomjs/neo#20328.
- Required-context binding for Engine lints, which is neomjs/neo#17171.

## Avoided Traps

- **Dropping checks to save slots:** every check stays; only the job count changes.
- **A matrix:** that is still ten jobs.
- **Stopping at the first red:** an author would fix one check per push. `!cancelled()` runs all nine.
- **Folding `Skills materialized` in:** it runs the pull request's install, and inside a shared job that code could rewrite the environment or the guards of every later step.
- **Ten commit statuses from one job:** that needs `statuses: write`, against #15's least-privilege contract, and a fork PR's read-only token can't post them, so neo's required contexts would block every external contribution.
- **Running only `pr-body` on `edited`:** with one aggregate context, it would overwrite a failing guard's red on the same SHA.

## Related

neomjs/neo#20328 (parent outcome) · #14 (PR governance across repositories) · #15 (the split this revisits) · #116 (tag pins) · #80 (baseline coordinates)

Sweeps at 2026-11-05T15:19Z found no equivalent:
- the latest open here and in neo;
- an all-state search on baseline / one job / runner / required contexts;
- A2A claims since 14:40Z;
- Memory Core on the problem nouns.

Owner: @neo-opus-grace, self-assigned after the peer decision. The neo ruleset swap is the one human-owned step.
Origin Session ID: 8545cff9-f4d7-4731-b93e-7dfad1347c8a
Retrieval Hint: "PR baseline ten jobs two jobs runner slots org-wide 20 required contexts aggregate isolated install"


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
- 2026-11-05T19:22:31Z @neo-opus-grace cross-referenced by #20328
### @neo-opus-grace - 2026-11-05T20:28:33Z

**Decision: one job, one aggregate context `baseline / PR baseline`** (Grace, 2026-11-05 ~20:30Z). This is a maintainer call, so it no longer waits on the operator's pick. Emmy authored #15's one-job-per-check design and has the signal on it.

Measured just now:
1. **Only neo requires the baseline contexts.** neo's `dev` ruleset requires all ten `baseline / *` contexts. Brain, Institution, Skills and devindex require none, and neo-stage has no rulesets. The swap is one ruleset edit in one repository.
2. **We hold `maintain`, not admin, on every consumer.** The operator applies that one swap when neo's pin bump merges. It is the only human step.
3. **The ten-statuses alternative fails on forks.** It needs `statuses: write`, against #15's least-privilege contract. A fork PR's token is read-only, so it couldn't post them at all, and neo's required contexts would block every external contributor's PR.
4. **Gating doesn't change.** All ten are required today, so one aggregate gates exactly as ten contexts do. Per-check results stay visible in the step names and the aggregate step's summary.

**Plan:** I take this ticket and build the one-job workflow:
- every check is a step with `continue-on-error`;
- a final step fails the job and names every failed check;
- on `edited`, only `pr-body` runs, the finding from the comment above;
- the test script is updated and the version bumped.

neo's pin-bump PR carries the migration note, and at its merge the operator swaps the ten contexts for the one. The other consumers only bump their pin.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 28a51314-b031-40b3-af35-6cc764b8cc90


- 2026-11-05T20:28:34Z @neo-opus-grace assigned to @neo-opus-grace
### @neo-opus-grace - 2026-11-05T20:30:25Z

**Correction to the decision above: two jobs, not one** (Grace, after reading the workflow line by line)

`Skills materialized` is the only check that runs the pull request's own code:
- `npm ci` runs the consumer's root lifecycle scripts;
- `npx --no-install neo-agent-skills-materialize` runs a binary resolved from the pull request's lockfile.

Inside a shared job, that code could append to `GITHUB_ENV` (for example `BASH_ENV`) or `GITHUB_PATH`, or rewrite the guards installed in `runner.temp`. It could then turn every later step green, the aggregate verdict included. Today each guard sits in its own job, so that can't happen. It is the same invariant the `pr-body` job's comment states: a diff cannot weaken the gate that judges it.

The other nine run no pull-request code. They read files, git history, the live body or this workflow's own `package.json`, and they run guards installed from the registry into `runner.temp`. They can share one job.

**Corrected shape:**
- `PR baseline` runs `version`, `release-ref`, `pr-base`, `source-comment-archaeology`, `substrate-size`, `npm-overrides`, `secrets`, `commit-authorship` and `pr-body` as steps. It does one checkout of the exact head with full history, and a final step fails the job while naming every failed check. On `edited`, only the `pr-body` steps run.
- `Skills materialized` stays its own job, unchanged.

That takes 10 jobs to 2, saving 8 runner slots per push instead of 9. The ruleset swap in neo replaces nine of its ten `baseline / *` contexts with `baseline / PR baseline` and keeps `baseline / Skills materialized`. Everything else in the decision stands.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 28a51314-b031-40b3-af35-6cc764b8cc90


- 2026-11-05T20:44:28Z @neo-opus-grace cross-referenced by PR #283
### @neo-opus-grace - 2026-11-05T22:46:19Z

**0.1.90 consumer map** (AC-5). Read on 2026-11-05 at each repository's `dev`. The README carries the procedure; this note carries the per-repository facts.

| Consumer | How it uses the package | Pin today | What moving to 0.1.90 takes |
|---|---|---|---|
| `neomjs/neo` | calls `reusable-pr-baseline.yml` as job `baseline` (`.github/workflows/pr-baseline.yml`) | `@v0.1.84` | A pin PR. The `dev` ruleset requires the ten old contexts (`baseline /` Commit authorship, PR base, PR body, Release ref, Secrets, Skills materialized, Skills version, Source comment archaeology, Substrate size, npm overrides). The repository administrator swaps them for `baseline / PR baseline` and `baseline / Skills materialized` in step with that PR's merge. |
| `neomjs/neo-agent-brain` | calls it as job `baseline` (`.github/workflows/shared-pr-baseline.yml`) | `@v0.1.84` | A pin PR only. Its `dev` rulesets require no `baseline / *` context. |
| `neomjs/neo-agent-institution` | calls it as job `baseline` (`.github/workflows/shared-pr-baseline.yml`) | `@v0.1.84` | A pin PR only. Its `dev` rulesets require no `baseline / *` context. |
| `neomjs/devindex` | calls it as job `baseline` (`.github/workflows/shared-pr-baseline.yml`) | `@v0.1.84` | A pin PR only. Its `dev` rulesets require no `baseline / *` context. |
| `neomjs/benchmarks`, and private repositories that install the package inside their own workflows | an npm pin, no call to this workflow | their own | Nothing: 0.1.90 changes only the reusable workflow. |

- **Old pins:** a caller at `@v0.1.84` keeps that tag's ten jobs until its own pin moves.
- **How the requirements were read:** `gh api repos/neomjs/<repo>/rules/branches/dev`. Classic branch protection is not visible to that read, so the Brain, Institution and devindex administrators should confirm it requires none of the old contexts before their pin PRs merge.
- **How the callers were found:** each repository's workflow files were read directly. Code search had missed devindex. `neo-agent-skills` itself runs the contract suite and never calls the workflow.
- **Ownership (corrected 2026-11-05):** I had said I'd open the four pin PRs. All four repositories run Dependabot's `github-actions` updates daily, and moving the tag is exactly what it is for, so its tag bumps are the pin PRs. My own would only duplicate them. 0.1.90 is published and tagged `v0.1.90`. When neo's Dependabot PR appears, the ruleset swap goes in step with its merge, owned as before by the post-merge row in PR #283.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 72d95313-a83c-478f-b64e-5266f2552939


- 2026-11-05T22:49:57Z @neo-opus-grace referenced in commit `378df2a` - "fix(workflows): keep the baseline's own reads out of the caller tree; pin exact gates and a command boundary (#281)

RA-1: the release version is read over the API and the team roster is cloned
into runner.temp, so the caller head is the PR-baseline job's only checkout and
no tracked path is replaced or deleted before a guard reads it. The contract
counts the checkouts and forbids file-moving commands, and a collision control
runs the secrets guard on a candidate tracking a credential under both former
paths: rejected as it arrived, hidden by the pre-fix deletion.

RA-2: every gate after the first is matched whole (`!cancelled()` exactly, or
one of four named exceptions), and Node runs only the workflow's stdin scripts
with no relative path executed; `!cancelled() && false` and `node ./attacker.mjs`
are negative controls.

RA-3: the README carries the migration procedure; the per-consumer map is on
#281."
- 2026-11-05T22:57:51Z @tobiu referenced in commit `2a17d17` - "ci(baseline): the PR baseline runs as two jobs, the guards in one and the caller's install alone (#281) (#283)

* ci(baseline): the PR baseline runs as two jobs, the guards in one and the caller's install alone (#281)

Nine checks that execute no pull-request code become steps of one
PR baseline job: one checkout, one guard install, every step after the
first under !cancelled(), no continue-on-error. Skills materialized
keeps its own job because it runs the caller's install. 10 jobs -> 2.

* fix(workflows): keep the baseline's own reads out of the caller tree; pin exact gates and a command boundary (#281)

RA-1: the release version is read over the API and the team roster is cloned
into runner.temp, so the caller head is the PR-baseline job's only checkout and
no tracked path is replaced or deleted before a guard reads it. The contract
counts the checkouts and forbids file-moving commands, and a collision control
runs the secrets guard on a candidate tracking a credential under both former
paths: rejected as it arrived, hidden by the pre-fix deletion.

RA-2: every gate after the first is matched whole (`!cancelled()` exactly, or
one of four named exceptions), and Node runs only the workflow's stdin scripts
with no relative path executed; `!cancelled() && false` and `node ./attacker.mjs`
are negative controls.

RA-3: the README carries the migration procedure; the per-consumer map is on
#281."
- 2026-11-05T22:57:52Z @tobiu closed this issue

