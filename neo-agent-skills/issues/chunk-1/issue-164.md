---
id: 164
title: A publish that starts seconds after its merge can find no PR yet and skip the release
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-10-10T18:46:01Z'
updatedAt: '2026-10-10T19:23:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/164'
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
closedAt: '2026-10-10T19:23:23Z'
---
# A publish that starts seconds after its merge can find no PR yet and skip the release

## Context

On 2026-10-10 the merge of #162 (0.1.36) ran `publish.yml` (run 38071670039). Its `origin` job read `repos/…/commits/02aa69b75d…/pulls` six seconds after the merge and got no PR, so `release-origin.mjs` failed closed ("expected one exact merged PR, found 0") and `publish` was skipped. Minutes later the same read returned #162 merged, and a re-run (attempt 2) published 0.1.36. The other five publishes that day succeeded. Defect-noted on A2A (`c4727606`).

## The Problem

GitHub indexes a merge commit's PR association asynchronously. The `origin` step reads it exactly once, so whether a release publishes depends on how fast the runner starts. The failure is the safe direction (nothing publishes without provenance), but each miss needs a human re-run.

## The Architectural Reality

- `.github/workflows/publish.yml`, job `origin`, step *Classify the exact merged PR*: one `gh api --paginate --slurp …/commits/${GITHUB_SHA}/pulls | node scripts/release-origin.mjs …`.
- `scripts/release-origin.mjs` is a pure classifier over the receipts it is given. It throws on zero or several exact matches and on an inconsistent author; it needs no change.
- `scripts/test-release-origin.mjs` pins the step's command shape with regexes.

## The Fix

The step retries the read and classification together, up to six attempts ten seconds apart, writes `publish=…` only on success, and exits 1 after the last failure. Retrying a genuine ambiguity costs a minute and still fails closed. The test runs the step's own `run:` block under bash, with stub `gh` and `sleep` on PATH, for "empty twice, then found" and "never found".

## Acceptance Criteria

- [ ] AC-1 A receipt that appears on the third read publishes; the step's output is `publish=true` (behavior test over the real run block).
- [ ] AC-2 A receipt that never appears exits 1 with no output, after six reads (same test).
- [ ] AC-3 The existing classifier and command-shape assertions still pass.
- [ ] AC-4 **Post-merge:** the next publishes succeed without a manual re-run.

## Out of Scope

The classifier's rules; the publish job's concurrency queue.

## Related

#162 (the miss), #14 (epic).

Sweeps: live latest-open sweep of this repository's 20 newest open issues at 18:45Z plus an org search for "release-origin", no equivalent · A2A: my defect-note `c4727606` is the only record · MC: none beyond today's · Own-assignment: none open.

Origin Session ID: afd79583-ff45-4f2c-96a5-08549257af36
Retrieval Hint: "publish release-origin found 0 commit pulls association race retry"

## Timeline

- 2026-10-10T18:46:01Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-10T18:46:02Z @neo-opus-grace added the `bug` label
- 2026-10-10T18:46:03Z @neo-opus-grace added the `ai` label
- 2026-10-10T18:46:03Z @neo-opus-grace added the `build` label
- 2026-10-10T18:47:41Z @neo-opus-grace cross-referenced by PR #165
- 2026-10-10T19:15:49Z @neo-opus-grace referenced in commit `481254b` - "fix(release): the publish origin step reads a late merge receipt again before failing closed (#164)"
- 2026-10-10T19:23:23Z @tobiu referenced in commit `01086d1` - "fix(release): the publish origin step reads a late merge receipt again before failing closed (#164) (#165)"
- 2026-10-10T19:23:23Z @tobiu closed this issue

