---
id: 427
title: The shared baseline reruns its guards on a body edit and repairs in a runner of its own
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-19T17:45:53Z'
updatedAt: '2026-11-19T21:10:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/427'
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
blocking:
  - '[ ] 21504 Fold File Size Guard into the shared baseline''s guards job'
closedAt: '2026-11-19T21:10:23Z'
---
# The shared baseline reruns its guards on a body edit and repairs in a runner of its own

## Context

CI has 20 concurrent runner jobs across the org (operator, 19 Nov 2026). In Engine a PR body edit starts 7 of them, merged PRs included: my post-merge checkbox tick on neomjs/neo#21468 started PR Baseline (3 jobs), File Size Guard, the claim responder and 2 same-head repairs. Same-Head Check Repair ran 46 times in about two hours that afternoon. Agents edit bodies often: review responses, evidence, post-merge ticks.

**Narrowed at intake, 19 Nov 20:10Z, and again at 20:20Z (see Intake below):** only the merged-PR skip stays. Three items failed design-authority checks and moved to Out of Scope: the open-PR edit skip broke the workflow's one-context rule, the repair fold needs a permission every caller would have to grant, and giving guards the live body breaks the job split's least-privilege rule.

## The Problem

On every event the shared baseline runs all three of its jobs, and a merged PR's body edit runs the same set, though nothing can merge on that head again.

## The Architectural Reality

- `.github/workflows/reusable-pr-baseline.yml`: `pr-baseline` (`contents: read`, `pull-requests: read`) fetches the live body and runs the body checks; `skills-materialized` runs the caller's install; `guards` (`if: inputs.guards != '[]'`) runs the caller's own guard scripts.
- The header's design rule (line 28): "Every event runs every check: one context carries every verdict, so an edit that ran only the body checks would overwrite another check's red on the same head."
- The job split (lines 109–110, 429–431): `pull-requests: read` is "Granted on this job, which runs no pull-request code"; the caller's guards "are pull-request code and stay out of `PR baseline`".

Design authority: "Every event runs every check: one context carries every verdict, so an edit that ran only the body checks would overwrite another check's red on the same head." (`reusable-pr-baseline.yml:28`). The verdict it protects belongs to a head that can still merge, and a merged head cannot. This change amends that header with the merged-PR exception in the same diff.

## The Fix

1. All three jobs skip when the event's pull request is merged (`github.event.pull_request.merged`). A merged head cannot merge again, so a skipped (successful) check there hides nothing.
2. The version is bumped.

Decision Record impact: none.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `reusable-pr-baseline.yml`: all three jobs | the operator's 20-slot rule, 19 Nov | skipped on an event whose pull request is merged | an open or closed-unmerged PR runs every check, as the header's rule requires | the workflow header | contract arms in `scripts/test-reusable-pr-baseline.mjs` |

## Acceptance Criteria

- [ ] AC-1: on an event whose pull request is merged, none of the three jobs runs; an open PR's edit still runs all three. Contract arms read the workflow.
- [ ] AC-2: the version is bumped.

## Post-Merge Validation

- [ ] After Engine's pin bump, a body edit on a merged Engine PR starts no baseline job (jobs API).

## Out of Scope

- **Skipping the file jobs on an open PR's edit.** This breaks the header's rule quoted above. A job skipped by a condition reports success ("Successful check statuses are `success`, `skipped`, and `neutral`", GitHub's required-status-check troubleshooting page), and GitHub documents no rule for which of two same-named check runs on one commit a required check reads. An edit after a real `guards` red would put a passing `guards` beside it. Avoiding it means a second caller without `edited` and new required contexts in every consumer's ruleset.
- **Folding the same-head repair into `pr-baseline`** (neomjs/neo#20064's promised upstream home). Re-running runs needs `actions: write`. Every caller would have to grant it, or its baseline is rejected before any job starts, so it is a consumer-contract change that waits on that decision. Until then the repair stays in each consumer.
- **Giving the `guards` job the live body and the base** (for Engine's File Size Guard, neomjs/neo#21504). The job runs pull-request code, so it may not hold `pull-requests: read` (the job split above). The alternatives are a job output from `pr-baseline` or the event payload's body, and a re-run of failed jobs serves either one stale, which defeats the repair's re-judgment of a corrected body. Keeping File Size Guard in Engine is the third option. This needs a decision before anything is built.
- **A closed but unmerged PR.** It can be reopened, and `reopened` is not a caller type, so skipped checks there could read green on reopen.
- The CI-jobs justification check: #426.

## Related

neomjs/neo#20064 · neomjs/neo#21504 · #281 · #318 · #426 · #428

Live sweep at 2026-11-19T17:37Z: I read the latest 20 open issues here and searched open and closed issues for same-head, check repair and runner jobs. Backlog: 40 open (orange). A2A: no claim. My own assignments: #426 is the sibling, next in line.

Intake, 19 Nov 20:10Z (earlier-session author). The drift probe fired: #428 changed `reusable-pr-baseline.yml`, `scripts/test-reusable-pr-baseline.mjs` and `package.json`, so the full gate ran. Prescription checked: `.github/workflows/reusable-pr-baseline.yml` owns the concern. The ticket's reality: same-day, no stale band, no newer artifact covers it. Three items failed design-authority checks (Out of Scope above; the third was caught at 20:20Z, reading the `guards` job before building it), and the Contract Ledger was missing and is added. Verdict: needs-narrowing, narrowed.

Ownership: self-assigned.

Origin Session ID: 41be8b70-720d-4106-828c-036c564b0fb7
Retrieval Hint: "shared PR baseline: merged-PR edits skip every job; open-PR edit skip, repair fold and guard body access each fail a design rule"


## Timeline

- 2026-11-19T17:45:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-19T17:45:55Z @neo-opus-grace added the `enhancement` label
- 2026-11-19T17:45:56Z @neo-opus-grace added the `ai` label
- 2026-11-19T17:45:56Z @neo-opus-grace added the `performance` label
- 2026-11-19T17:45:56Z @neo-opus-grace added the `build` label
- 2026-11-19T17:46:02Z @neo-opus-grace cross-referenced by #426
- 2026-11-19T17:46:32Z @github-actions added the `needs-design-authority` label
### @neo-opus-grace - 2026-11-19T18:29:19Z

**Session handover (19 Nov, 18:25Z) — owner: @neo-opus-grace, not started.**

- **Blocked on:** #428 (#426) merging. Both edit `reusable-pr-baseline.yml`, so they would conflict in parallel.
- **Plan:** branch from `dev` after #428 lands.
  1. `skills-materialized` and `guards` skip on `edited`, and all three jobs skip on a PR that is not open.
  2. A caller input `body_guards` runs the caller's body-reading guards inside `pr-baseline` (Engine's `check-file-sizes --pr-body-file`).
  3. `pr-baseline` ends with the same-head repair: re-run earlier red runs of its own workflow on this head after a green, by run number, as in neomjs/neo#20064's `same-head-check-repair.yml`.
  4. Pin each piece with contract arms and mutations in `scripts/test-reusable-pr-baseline.mjs`.
  5. Version bump.
- **Engine half:** neomjs/neo#21500 retires `file-size-guard.yml` and `same-head-check-repair.yml` after the pin bump.

- 2026-11-19T18:48:40Z @neo-opus-grace marked this issue as blocking #21504
- 2026-11-19T19:10:50Z @neo-gemini-julia cross-referenced by PR #428
- 2026-11-19T20:23:48Z @neo-opus-grace cross-referenced by PR #429
- 2026-11-19T20:47:34Z @github-actions removed the `needs-design-authority` label
- 2026-11-19T21:10:23Z @tobiu referenced in commit `b33e694` - "feat(ci): the shared baseline starts no runner on a merged pull request's events (#427) (#429)

Each of the three jobs skips when the event's pull request is merged: its head cannot merge again, so a skipped check there hides nothing. Open and closed-unmerged pull requests still run every check, as the header's one-context rule requires, and the contract refuses any job gated on the event action. Version 2.3.0."
- 2026-11-19T21:10:24Z @tobiu closed this issue
- 2026-11-19T21:17:29Z @neo-opus-vega cross-referenced by PR #1867

