---
id: 436
title: A declared guard can read neither the PR's base nor its live body
state: OPEN
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-22T03:36:51Z'
updatedAt: '2026-11-22T03:36:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/436'
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
blocking:
  - '[ ] 21504 Fold File Size Guard into the shared baseline''s guards job'
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
2. The checkout takes `fetch-depth: 0`, so the base is in history, and `persist-credentials: false`, so git keeps no token for later steps.
3. Before `npm ci`, the same `Read the live pull-request body` step that `pr-baseline` runs, writing `${{ runner.temp }}/pr-body.md`.
4. `Run the declared guards` passes every guard `BASE_SHA` (`github.event.pull_request.base.sha`) and `BODY_FILE`.
5. The comments on both jobs state the rule as its real invariant: no job that runs pull-request code holds more than read access to its own repository. A minor release follows, because callers gain inputs and change nothing.

## Acceptance Criteria

- [ ] AC-1: A guard declared by a fixture caller reads `BODY_FILE` and finds the live body, and `git cat-file -e "$BASE_SHA"` succeeds in the job.
- [ ] AC-2: The `guards` job fetches the body itself on every run attempt, from no job output and no event payload.
- [ ] AC-3: The `guards` checkout persists no credentials.
- [ ] AC-4: `package.json` carries the minor version bump.

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

