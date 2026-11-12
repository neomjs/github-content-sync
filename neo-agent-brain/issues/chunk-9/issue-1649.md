---
id: 1649
title: Brain Unit says when a re-run re-compared the old suite reports
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - github_actions
assignees: []
createdAt: '2026-11-12T14:51:09Z'
updatedAt: '2026-11-12T14:51:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1649'
author: neo-opus-ada
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
# Brain Unit says when a re-run re-compared the old suite reports

## Context

`Brain Unit`'s `unit` job compares two report artifacts that `suite (head)` and `suite (base)` upload. Both suite jobs always succeed (`continue-on-error: true`, because their report is their output), so GitHub's "Re-run failed jobs" re-runs `unit` alone. It downloads the first attempt's reports again and repeats their verdict. A seat that re-runs a suspected flake that way sees the same `1 introduced` and reads it as a reproduction.

Sightings, across three maintainer seats:
- PR #1355 and PR #1361, 2026-10-29. On #1361 (run 40909604135), attempts 1 and 2 compared the same suite reports.
- PR #1449 (2026-11-03) and PR #1477 (2026-11-05): a red `unit` gate that turned out to be a re-comparison, each cleared by a full re-run.
- PR #1645, 2026-11-12, run 43230881593: attempts 2 and 3 re-compared attempt 1's reports. The resulting "2 of 2" went into two defect-notes before a full re-run came back green. The underlying flake is #1647.

## The Problem

Nothing in the job's output tells a fresh sample from a re-comparison. The suite jobs' completion times reveal it, but only to a reader who already knows to look, and the trap has caught readers who did know.

## The Architectural Reality

- `.github/workflows/brain-unit.yml`: `suite` uploads `unit-report-<side>`, which holds `test-results.json` only. `unit` downloads `unit-report-*` and runs `node test/playwright/compareUnitFailures.mjs --head … --base …`.
- `test/playwright/compareUnitFailures.mjs` prints the failure diff and ends with `compareUnitFailures: <n> introduced, <n> fixed, <n> pre-existing`.
- Design authority: n/a. The comparison's verdict is right; this adds its provenance.

## The Fix

1. `suite` writes `${{ github.run_attempt }}` beside its report and uploads both files.
2. `unit` names the attempt each report came from, next to its own `${{ github.run_attempt }}`. When the head report is older than the `unit` job, the output ends with one line: the verdict re-compares attempt `<n>`'s reports and is not a new sample, so re-run all jobs (`gh run rerun <run id>`, without `--failed`) to sample again.
3. The verdict itself does not change. A `unit` job that failed for an infrastructure reason can still be re-run alone.

## Acceptance Criteria

- [ ] AC-1: a first-attempt run names both reports' attempt, equal to its own, and adds no warning.
- [ ] AC-2: "Re-run failed jobs" on a red `unit` prints the re-comparison line, naming the earlier attempt and the full re-run command. Shown on a real re-run of the PR's own workflow.
- [ ] AC-3: "Re-run all jobs" names the new attempt for both reports again, with no warning.
- [ ] AC-4: if the line is built in `compareUnitFailures.mjs`, a unit arm covers both cases.

## Out of Scope

- Failing the job on a re-comparison. That would block re-running a crashed `unit` on its own.
- Other workflows. None in this repository is known to share the fan-in shape; the intake confirms it with a `download-artifact` sweep of `.github/workflows/`.

## Related

#1645 · #1647 · #201 (the workflow's origin; its Out of Scope excludes nothing here)

Live latest-open + closed sweep: latest open issues (157 open, red band; no owning open ticket for this workflow) and closed/all searches (`re-run failed jobs`, `rerun unit compare`, `run_attempt`, `compareUnitFailures`, `re-comparison`, plus an org-wide `"re-run failed"` search) at 2026-11-12T14:52Z; no equivalent or exclusion found. A2A in-flight claim sweep: the latest 30 messages, all read states; no claim on this scope. MC sweep: the sightings above, never filed. Own-assignment sweep: none on this surface.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. A fork PR that changes `.github/workflows/` has its runs approved after a maintainer reads the diff, and a maintainer can trigger the AC-2 re-run.

Retrieval Hint: "brain-unit re-run failed jobs re-compares old suite reports same verdict not a reproduction run_attempt"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T14:51:11Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T14:51:11Z @neo-opus-ada added the `good first issue` label
- 2026-11-12T14:51:12Z @neo-opus-ada added the `help wanted` label
- 2026-11-12T14:51:12Z @neo-opus-ada added the `ai` label
- 2026-11-12T14:51:12Z @neo-opus-ada added the `github_actions` label

