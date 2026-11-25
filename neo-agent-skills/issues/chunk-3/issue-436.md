---
id: 436
title: A declared guard can read neither the PR's base nor its live body
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-22T03:36:51Z'
updatedAt: '2026-11-25T16:20:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/436'
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
  - '[x] 21504 Fold File Size Guard into the shared baseline''s guards job'
closedAt: '2026-11-25T16:06:55Z'
---
# A declared guard can read neither the PR's base nor its live body

## Context

neomjs/neo#21504 folds File Size Guard into this baseline's `guards` job. That saves two runner jobs per event of an Engine PR the guard watches: its own job and its Same-Head Check Repair run, out of the organization's 20 concurrent runner slots (the capacity context is on neomjs/neo D#21725). The guard diffs against the base and reads the live PR body for size justifications. #427 recorded the choice of how a guard gets that body as a decision to make before building (its Out of Scope), then closed without it. I wrote the `pr-baseline`/`guards` split (#281, #318), so the decision is mine to take, and this ticket takes it.

## The Problem

At `dev` `a73c269` and in the released `v2.3.0`, the `Run the declared guards` step passes only `GUARDS`. The job checks out shallow, and its token has `contents: read` alone. A guard gets neither the base commit nor the PR body, so `check-file-sizes` cannot run there.

## The Architectural Reality

Design authority: "After neo-agent-skills#427 it gives each guard the base's history and the live body (`BASE_SHA`, `BODY_FILE`)." (neomjs/neo#21504, The Architectural Reality)

- `pr-baseline` reads the live body with `pull-requests: read` (`reusable-pr-baseline.yml:111-115`), "Granted on this job, which runs no pull-request code".
- `guards` runs pull-request code: `npm ci` and the guards themselves (`:431-447`). Its checkout persists the job token in `.git/config` (the `actions/checkout` default), so that code already holds a token with `contents: read`.
- There are four ways to hand a guard the body:
  1. A job output or artifact from `pr-baseline`. It serializes the two jobs, and "Re-run failed jobs" reuses the earlier attempt's output, so a corrected body is judged stale. That defeats the same-head repair (neomjs/neo#20064).
  2. The event payload's body: stale on a re-run for the same reason.
  3. File Size Guard stays in Engine, and both runner jobs stay.
  4. `guards` reads the body itself with `pull-requests: read`: live on every run and re-run, with no serialization.
- **The cost of option 4.** The token that pull-request code can reach gains `pull-requests: read` on the same repository. Step order is no barrier, because code in a job can read secrets from the runner process's memory (the 2025 `tj-actions/changed-files` compromise, CVE-2025-30066). On a fork PR the token is read-only and the data is public. On a same-repository PR the author already has push access. So the PR metadata this adds is no more sensitive than the contents the token already reads. Guards are the author's own code by design (`:431-433`), and the author controls the body as well, so a body a guard reads adds no trust problem.

## The Fix

Option 4, in the `guards` job of `reusable-pr-baseline.yml`:

1. `permissions: {contents: read, pull-requests: read}`.
2. The checkout takes `fetch-depth: 0`, so the base is in history, and `persist-credentials: false`, so git keeps no token for later steps. A `Base commit in history` step asserts `git cat-file -e "${BASE_SHA}^{commit}"` before any guard runs.
3. Before `npm ci`, the same `Read the live pull-request body` step that `pr-baseline` runs, writing `${{ runner.temp }}/pr-body.md`.
4. `Run the declared guards` passes every guard `BASE_SHA` (`github.event.pull_request.base.sha`) and `BODY_FILE`.
5. The comments on both jobs state the rule as its real invariant: no job that runs pull-request code holds more than read access to its own repository. A minor release follows, because guards gain two variables and callers change nothing.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `BASE_SHA` and `BODY_FILE` in a caller's declared guards | `reusable-pr-baseline.yml`, `inputs.guards` description | Every guard reads the PR base commit (in history) and the live body as a file | A caller that declares no guards skips the job, unchanged | `inputs.guards` description; the `guards` job comment | `scripts/test-reusable-pr-baseline.mjs`: a fixture guard through the real runner, plus mutation arms |
| `guards` job permissions and checkout | This ticket's option 4; the `pr-baseline` grant | `contents: read` + `pull-requests: read`; full history; no persisted credential | None: the grant stays read-only on the job's own repository | Both jobs' comments | Same test: required contract plus a mutation arm per property |

## Acceptance Criteria

- [ ] AC-1: A fixture guard reads `BODY_FILE` and `BASE_SHA` through the `guards` runner, the job's body read is the same live read as `pr-baseline`'s, and the job asserts `git cat-file -e "${BASE_SHA}^{commit}"`.
- [ ] AC-2: The `guards` job fetches the body itself on every run attempt, from no job output and no event payload.
- [ ] AC-3: The `guards` checkout persists no credentials.
- [ ] AC-4: `package.json` carries the minor version bump.

## Post-Merge Validation

- [ ] The first real `guards` run on a release carrying this (neomjs/neo#21504's pull request) passes `Base commit in history`, and a guard there reads the live body. A reusable workflow runs only from a published release tag, so no pre-merge job can show it.

## Out of Scope

- Folding File Size Guard itself: neomjs/neo#21504, once a release carries this.
- Same-Head Check Repair's home (#427's Out of Scope).

## Related

neomjs/neo#21504 (blocked by this) · #427 (left the decision open) · #318 (the `guards` job) · #281 (the job split) · neomjs/neo#20064 (the same-head repair).

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 open Skills issues, plus open and closed searches for "guards body", `BODY_FILE`, "guards BASE_SHA", "guards pull-requests read" and "File Size Guard", at 2026-11-22T03:35Z. Hits read: #427 (its Out of Scope is this ticket's source), #281 and #318 (their Avoided Traps forbid pull-request code in `pr-baseline` and `statuses: write`, neither of which this needs). Open #14 and #80 are broader. No equivalent. Backlog: 42 open (orange), and no open owner to fold into.
A2A: no overlapping `[lane-claim]` or `[lane-intent]` in the latest 12 messages at 03:35Z.
MC sweep: 1 `query_raw_memories` query on the symptom; no rationale against this.
Own-assignment sweep: 4 open, none on the baseline.
Structure map: N/A, a workflow file only.

Owner: @neo-opus-grace, as the author of the job split and the `guards` job. It lands after the Engine 14 cut and is not a release item, so it carries no `good first issue` label.

Origin Session ID: fcc79e4e-58d3-45ee-a307-579657aa8948
Retrieval Hint: "shared baseline guards job live PR body BASE_SHA BODY_FILE pull-requests read File Size Guard fold re-run failed jobs stale"

## Timeline

- 2026-11-22T03:36:51Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-22T03:36:54Z @neo-opus-grace added the `enhancement` label
- 2026-11-22T03:36:54Z @neo-opus-grace added the `ai` label
- 2026-11-22T03:36:54Z @neo-opus-grace added the `build` label
- 2026-11-22T03:37:05Z @neo-opus-grace marked this issue as blocking #21504
- 2026-11-22T03:37:30Z @neo-opus-grace cross-referenced by #21504
- 2026-11-24T23:08:40Z @neo-opus-grace cross-referenced by #453
- 2026-11-25T14:18:18Z @neo-opus-grace cross-referenced by PR #468
- 2026-11-25T15:19:32Z @neo-opus-grace referenced in commit `0d2d247` - "chore(release): merge dev at 2.8.5 (#462) into the guards branch, version 2.9.0 (#436)

Resolves only the version-bearing files: dev's side, then npm version 2.9.0, which regenerated every plugin manifest including verify-before-code."
- 2026-11-25T15:51:58Z @neo-opus-grace referenced in commit `f4e2067` - "chore(release): merge dev at 2.8.7 (#465, #466) into the guards branch, version 2.9.0 (#436)

Resolves only the version-bearing files: dev's side, then npm version 2.9.0, which regenerated every plugin manifest."
- 2026-11-25T16:06:55Z @tobiu referenced in commit `486a4e1` - "feat(ci): the guards job hands every guard the base commit and the live PR body (#436) (#468)

A caller's guards now read BASE_SHA (in history: full-depth checkout, asserted
by a `Base commit in history` step) and BODY_FILE, written by the same live
read PR baseline runs, before `npm ci`. The job reads the body itself on every
run attempt, so "Re-run failed jobs" judges a corrected body, not the first
attempt's. It gains `pull-requests: read`, stays read-only on its own
repository, and its checkout persists no credential.

The contract test pins the body step byte-equal to PR baseline's, runs a
fixture guard through the real runner, and adds ten mutation arms.

Minor release: 2.9.0."
- 2026-11-25T16:07:07Z @tobiu closed this issue
### @neo-opus-grace - 2026-11-25T16:20:00Z

**Post-Merge Validation: done, on neomjs/neo#22206 (the first caller on v2.9.0).**

In PR Baseline run `45652262107`, `baseline / guards` (job `139339594384`) passed every step: checkout (full history), `Read the live pull-request body`, `Base commit in history`, and `Run the declared guards`. Engine's new `file sizes` guard ran on `$BASE_SHA` and `$BODY_FILE` and logged `check-file-sizes: 4 path(s) evaluated, 4 reported, 0 violations.`

**One caller broke, and it is fixed in that same PR.** The full-depth checkout makes `git fetch --unshallow` fatal ("on a complete repository", exit 128). Engine's `spec retirement` guard used that flag, so #22206 drops it. That is the first caller to move its pin to v2.9.0. I checked the other callers' workflows: neo-agent-brain, devindex and neo-agent-institution have no `--unshallow` and no credentialed git or `gh` in their guards. Their Dependabot bumps to v2.9.0 are safe.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 3a0cf37c-1c66-4dab-ba0f-9c29b72fb048

- 2026-11-25T16:33:25Z @neo-gpt-sophie cross-referenced by PR #22206

