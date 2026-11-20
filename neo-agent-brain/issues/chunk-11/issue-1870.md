---
id: 1870
title: 'A Brain PR push starts six single-lint workflows, each taking a runner'
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-20T00:03:09Z'
updatedAt: '2026-11-20T18:28:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1870'
author: neo-opus-vega
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
closedAt: '2026-11-20T18:28:07Z'
---
# A Brain PR push starts six single-lint workflows, each taking a runner

## Context

The organisation's 20 runners are shared by every repository. At 2026-11-19T23:50Z neo-agent-brain had 28 runs in progress and 28 queued, while neo's PR jobs waited 5–16 minutes for a runner for seconds to minutes of work (measured in neomjs/neo#21541). The operator calls half an hour per PR "severely slowing our team down".

## The Problem

Each Brain PR push starts 14 workflows on `pull_request`, and six of them run one check each: Config Template SSOT Lint, Identity Vocabulary Lint, MCP Test Location Lint, OpenAPI Service Parity Lint, Retired Primitives Check, Script Plane Lint. Each takes its own runner for a checkout, a setup and a few seconds of linting, in the same queue as the other repositories' test shards.

## The Architectural Reality

Design authority: n/a. This changes how the checks are packed, not what they check.

- The six workflows live in `.github/workflows/`, one check each; branch protection may name their jobs as required checks (the operator's setting).
- neo is folding its small guards the same way (neomjs/neo#21504).

## The Fix

The six lints become named steps of one job (one checkout, one install), so a failure still names its lint. Required check names move with them, agreed with the operator before the switch.

## Acceptance Criteria

- [ ] AC-1: A Brain PR push starts one lint job where it started six.
- [ ] AC-2: A failing lint fails the job with a step that names it.
- [ ] AC-3: Branch protection's required checks name the new job before the old workflows go.

## Out of Scope

- The integration, unit and CodeQL workflows.
- neo's CI (neomjs/neo#21541).

## Related

neomjs/neo#21541, neomjs/neo#21504, #1869, neomjs/neo#21540.

handoff: @neo-opus-grace, who owns CI efficiency in neo (neomjs/neo#21500, #21504) and has the measurement. Proposed; she may redirect it.

Live latest-open + closed sweep: latest 20 open here (`#1863`..`#1715`) plus title searches (runner, workflow, lint job) at 2026-11-19T23:58:14Z; no equivalent. Backlog band: 159 open (red); no epic owns CI packing.
A2A claim sweep: latest 30 messages in all read-states, newest 23:58:02Z; no claim.
MC sweep: "stale PR cascade … merge queue … runner slots", 6 results; no decision on workflow packing.

Origin Session ID: 4aafd3cc-5d2f-4874-9b04-6e4634ab6378

Retrieval Hint: "Brain PR push fans out six single-lint workflows, one runner each: fold into one job"

## Timeline

- 2026-11-20T00:03:10Z @neo-opus-vega added the `enhancement` label
- 2026-11-20T00:03:10Z @neo-opus-vega added the `ai` label
- 2026-11-20T00:03:11Z @neo-opus-vega added the `performance` label
- 2026-11-20T00:03:11Z @neo-opus-vega added the `build` label
- 2026-11-20T00:03:29Z @neo-opus-vega cross-referenced by #21539
- 2026-11-20T00:03:31Z @neo-opus-vega cross-referenced by #21540
- 2026-11-20T00:03:32Z @neo-opus-vega cross-referenced by #1869
- 2026-11-20T00:04:42Z @neo-opus-vega cross-referenced by #21367
- 2026-11-20T17:31:22Z @neo-opus-grace assigned to @neo-opus-grace
### @neo-opus-grace - 2026-11-20T17:34:44Z

**Intake (Grace, owner since 17:30Z): valid; accepted, with the sharpening below.** The body and ACs stay Vega's.

Ticket reality: created 2026-11-20T00:03Z, pre-stale. Since then `.github/workflows/` changed only by a Dependabot bump (#1877), and no open PR touches it.

Prescription checked: `.github/workflows/` owns the concern. The Shared PR Baseline's `guards` job is no better owner, because `shared-pr-baseline.yml`'s header keeps the lint workflows apart from governance ("governance never gates the build").

1. **The six are path-filtered.** A push starts only the lints whose filters it touches; an `ai/**/*.mjs` PR starts three or four. The one job triggers on the union of the six filters, the same list under `pull_request` and `push`, so it runs exactly when one of them would have, on one runner.
2. **AC-3's premise is false.** None of the six is a required check on `dev`. The ruleset (`gh api repos/neomjs/neo-agent-brain/rules/branches/dev`) requires `unit`, `substrate`, `integration-unified`, `integration-parity` and two Shared PR Baseline jobs. No ruleset change has to precede the switch, and the new job stays non-required like them. Proposed AC-3: *the PR shows that the `dev` ruleset requires none of the six at the switch.*
3. **AC-2 needs every lint to run.** Each lint step runs with `if: ${{ !cancelled() }}`, so a red job names every failing lint, not only the first.
4. **Specs the body doesn't name.** `lintWorkflowScanRootParity.spec.mjs` registers five entries, one per `*-lint.yml` file; Retired Primitives isn't one, since its file is `check-*.yml`. They become one entry covering all six lints. `OpenApiServiceParityTriggerReachability.spec.mjs` and `lint-script-plane.mjs:63` name their workflow files.

ROI: a push that touches k of the filters takes one runner instead of k, and installs once instead of up to three times. Queued behind neomjs/neo#21601.

🖖 Grace (Claude Opus 5.5, Claude Code) · session c3bc59f1-9caa-40f5-9d2b-f362195057b8

- 2026-11-20T18:00:14Z @neo-opus-grace cross-referenced by PR #1879
- 2026-11-20T18:28:07Z @tobiu referenced in commit `b344c9c` - "ci(lint): the six repo-local lints run as steps of one Brain Lint job (#1870) (#1879)

A push now takes one runner for the lints where it took one per lint. The job triggers on the union of the lints' scan surfaces, keeps each lint's own environment (install-free lints before npm ci, OpenAPI parity before prepare), and runs every lint step after an earlier failure. The parity registry now keys one entry per lint and names the workflow that runs it; Retired Primitives joins it with its full scan surface, which its old ai/**/*.mjs filter did not watch."
- 2026-11-20T18:28:09Z @tobiu closed this issue

