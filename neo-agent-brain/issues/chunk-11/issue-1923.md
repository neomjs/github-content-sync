---
id: 1923
title: Deduplicate superseded same-SHA workflow suites in headCiOf
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-25T08:39:38Z'
updatedAt: '2026-11-25T10:05:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1923'
author: neo-gemini-maryam
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
closedAt: '2026-11-25T10:05:38Z'
---
# Deduplicate superseded same-SHA workflow suites in headCiOf

Please comment to claim before starting; check current assignees and linked PRs.

## Context

On `2026-11-25`, `@tobiu` observed that [`neomjs/neo-agent-brain#1917`](https://github.com/neomjs/neo-agent-brain/pull/1917) (`fe6d3164af0eae2881150b74cb60a326f687ae9f`) had green CI checks and cross-family `APPROVED` review (`get_conversation(..., projection: 'merge-readiness')` -> `verdict: "merge-ready-observed"`, `strictMergeReady: true`, `blockers: []`, `mergeStateStatus: "CLEAN"`), yet Fleet Manager UI did not list it under `Awaiting Merge` (`awaitingMerge`) and `fleet-open-work` emitted a phantom `[open-work:repair-ci] ... failing: none in the producer snapshot` task (`MESSAGE:df340d51-38c1-410c-bf5f-d27a3f1dc8e6`).

## The Problem

On commit `fe6d3164af0eae2881150b74cb60a326f687ae9f` of `#1917`:
1. At `2026-11-24T15:48Z` (`pull_request: synchronize`), `Shared PR Baseline` (`W_kwDOUBzDFM4V5S_O`, `CheckRun` `CR_kwDOUBzDFM8AAAAgQ4BAVw`) failed (`FAILURE`) because the PR description contained raw HTML.
2. At `2026-11-25T00:29Z`, normalizing the PR description to Markdown triggered a **new** `CheckSuite` on the same head commit SHA (`pull_request: edited`) for the same workflow `W_kwDOUBzDFM4V5S_O` (run `45518456029`), which completed `SUCCESS`.
3. When a GitHub Actions workflow re-runs via a new workflow run (`pull_request: edited`, `reopened`, or `workflow_dispatch`) rather than an in-place job rerun, GitHub keeps **both** `CheckSuite`s attached to the commit OID and leaves `commit.statusCheckRollup.state` at `"FAILURE"` (even while GitHub branch rules evaluate the latest check-run per context and report `mergeStateStatus: "CLEAN"`).
4. `evidenceOf` in `ai/services/fleet/openWorkCiRepair.mjs` (`lines 108-119`) already deduplicates `statusCheckRollup.contexts` by `[kind, app.id, workflow.id, name]` (`latestOf` by `startedAt`), producing `failedChecks: []` (`"failing: none in the producer snapshot"`).
5. However, `headCiOf` in `ai/services/fleet/openWorkReducer.mjs` (`lines 139-141`) short-circuits to `return rollup` (`'red'`) before reaching its own `byWorkflow` deduplication loop (`lines 150-162`) whenever the commit has neither an informative `pull_request_review` check suite nor a `review-admission/mergeability` commit status — which is always true in `neomjs/neo-agent-brain`, `neomjs/neo-agent-skills`, `neomjs/neo-agent-institution`, and `neomjs/business`.
6. Because `row.ci` is `'red'`, `holderOf(row)` in `ai/services/fleet/openWorkHolder.mjs` (`line 279`) returns `{role: 'rotation', kind: 'repair-ci', ids: []}` instead of `{role: 'operator', kind: 'merge', ids: []}` (`line 334`), hiding the PR from `fleetOpenWorkSource.mjs` (`line 105`, `awaitingMerge`) and dispatching a false `repair-ci` wake.

## The Architectural Reality

- `Design authority: "and the newest run per workflow.id wins because checkSuites(last: N) returns nodes oldest first"` (`ai/services/fleet/openWorkReducer.mjs`, lines 115-117; matching `#1864`'s Contract Ledger: `"keeping the newest suite per workflow.id (so a superseded run of the same workflow on the same head is ignored)"`).
- `ai/services/github-workflow/queries/openWorkQueries.mjs` (`lines 218-274`) already fetches `commit.status { state contexts { context state } }` (`'status' in commit` is `true` even when `commit.status === null`) and `commit.checkSuites(last: 100) { pageInfo { hasPreviousPage } nodes { status conclusion workflowRun { event workflow { id } } } }` on both `OPEN_WORK_SNAPSHOT` and `OPEN_WORK_OUTSIDE_SNAPSHOT` at GraphQL formula cost `5` / `6`.
- `ai/services/fleet/openWorkReducer.mjs` (`lines 139-162`):
  - `#1864`'s unit test (`test/playwright/unit/ai/services/fleet/openWorkReducer.spec.mjs`, lines `765-777`) tested superseded `WF_BASE` (`FAILURE` followed by `SUCCESS`) alongside a queued `pull_request_review` suite (`WF_REV`), which satisfied `suites.nodes.some(isInformativeReviewSuite)` and masked the early return at lines `139-141` when no `pull_request_review` suite or `review-admission/mergeability` status is present.

## The Fix

1. **`ai/services/fleet/openWorkReducer.mjs` (`headCiOf`):**
   - Add a pure helper `hasSupersededWorkflowSuite(nodes)` returning `true` when two or more non-`pull_request_review` `CheckSuite` entries in `suites.nodes` share a non-empty `workflowRun?.workflow?.id`.
   - Include `hasSupersededWorkflowSuite(suites.nodes)` in the `hasStatusField` entry condition (`lines 139-140`) so `headCiOf` runs its existing `byWorkflow` deduplication + `statusCi` aggregation whenever `suites.complete` is `true`, `'status' in commit` is `true`, and `suites.nodes` contains superseded runs of the same workflow — even when neither `pull_request_review` suites nor `review-admission/mergeability` statuses exist.
   - Update `headCiOf`'s `@summary` JSDoc to reflect superseded same-workflow `CheckSuite` deduplication alongside excluded review suites and `review-admission/mergeability`.
2. **Unit tests (`test/playwright/unit/ai/services/fleet/openWorkReducer.spec.mjs`):**
   - Add red-first unit coverage proving:
     - When `commit.status` is `null`, `statusCheckRollup.state` is `'FAILURE'` (or `'PENDING'`), no `pull_request_review` suite or `review-admission/mergeability` status exists, and `checkSuites` (`hasPreviousPage: false`) has an earlier `'FAILURE'` / `'CANCELLED'` suite followed by a newer `'SUCCESS'` suite for the same `workflow.id` (with all other head suites `'SUCCESS'` / `'SKIPPED'`), `normalizePullRequest` computes `ci: 'green'`, `holderOf` returns `{role: 'operator', kind: 'merge', ids: []}` on an approved head, and `planOpenWorkWakes` emits zero `repair-ci` wakes.
     - Conversely, when the **newest** suite for that `workflow.id` is `'FAILURE'` or `'IN_PROGRESS'`, or a sibling workflow / commit status is failing or pending, or `checkSuites` is truncated (`hasPreviousPage: true`), `ci` remains `'red'` or `'pending'` (fail-closed).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `normalizePullRequest` / `headCiOf` (`ai/services/fleet/openWorkReducer.mjs`) | `openWorkReducer.mjs:115-117`, `#1864`, `#759` | Enter `byWorkflow` newest-suite-wins deduplication when `suites.complete && 'status' in commit` and `suites.nodes` contains multiple non-review suites sharing a `workflow.id`, even without `pull_request_review` suites or `review-admission/mergeability` statuses | Preserve exact `CI_STATES[statusCheckRollup.state]` fallback when `rollup === 'green'`, `!suites.complete`, legacy fixtures omit `commit.status` or `suite.status`, or none of the three rollup-divergence conditions holds | JSDoc on `headCiOf` | `openWorkReducer.spec.mjs` |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1: `headCiOf` in `ai/services/fleet/openWorkReducer.mjs` resolves `ci: 'green'` when `suites.complete` is `true`, `'status' in commit` is `true`, and an earlier failing/cancelled `CheckSuite` is superseded on the same head SHA by a newer `COMPLETED`/`SUCCESS` `CheckSuite` of the same `workflowRun.workflow.id` (with all other non-review head suites and non-admission commit statuses passing), without requiring `pull_request_review` suites or `review-admission/mergeability` statuses and without changing `OPEN_WORK_SNAPSHOT` / `OPEN_WORK_OUTSIDE_SNAPSHOT` GraphQL cost (`5` / `6`).
- [ ] AC-2: `headCiOf` preserves fail-closed `'red'` / `'pending'` results when the newest suite for that `workflow.id` is failing or incomplete, when any other head suite or non-admission `StatusContext` is failing or pending, or when `checkSuites` pagination is incomplete (`hasPreviousPage: true`).
- [ ] AC-3: Unit specs in `test/playwright/unit/ai/services/fleet/openWorkReducer.spec.mjs` reproduce the `#1917` false-red shape (red on `dev`, green with the fix) and verify `holderOf` (`{role: 'operator', kind: 'merge'}`) plus zero `repair-ci` wakes.

## Out of Scope

- Deduplicating `workflowRun: null` third-party GitHub App check suites (would require adding `app { id }` to `checkSuites.nodes`; GitHub Actions workflows always populate `workflowRun.workflow.id`).
- Changing `PullRequestService.mjs` `checksVerdict` (`selectedChecksGreen` vs `SKIPPED` non-required checks; `checksGreen` and `validateMergeReady` already evaluate `strictMergeReady: true` on `#1917`).

## Avoided Traps

- **Using `mergeStateStatus === 'CLEAN'` as a shortcut to force `ci = 'green'`:** unsafe in repos without required status checks on the target branch, where `mergeStateStatus` can report `'CLEAN'` while non-required test workflows are actively failing or running.
- **Paginating `statusCheckRollup.contexts` on seat `OPEN_WORK_SNAPSHOT` queries:** unnecessary and inflates GraphQL cost; `checkSuites(last: 100)` with `workflowRun.workflow.id` is already fetched on both snapshot queries.

## Related

- #759
- #1864
- #1899
- #1917

---

Live latest-open + closed sweep: checked latest 20 open of 157 (`red` band; `#1920`..`#1741`) + open and closed searches (`headCiOf`, `checkSuites`, `statusCheckRollup`, `awaitingMerge`; `Out of Scope` and `Avoided Traps` read on `#1864` and `#1899`) at `2026-11-25T08:35Z`; no equivalent or prior exclusion found.
A2A in-flight claim sweep: latest 30 messages across all read-states (`2026-11-25T08:37Z`); no overlapping claim.
MC sweep: `"FM UI awaitingMerge merge-ready statusCheckRollup Shared PR Baseline edited rerun"` (`5` results) + `ask_knowledge_base` (`type: 'ticket'`, `5` refs), no prior exclusion found (`#759` notes `statusCheckRollup.state` staying `FAILURE` after a re-triggered check passes).
Own-assignment sweep: `0` open in `neomjs/neo-agent-brain`, none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): owning file `ai/services/fleet/openWorkReducer.mjs` (`ai/services/fleet/`); no new `.mjs` files.

unowned-rationale: Ready for a contributor or peer maintainer while the filing seat drains active `neomjs/neo` release-gate review/RC items (`#22179`, `#22168`).

Origin Session ID: f56f17b9-b599-466c-8afb-8f8a3dc70c36
Retrieval Hint: "FM UI awaitingMerge missing merge-ready pull_request edited Shared PR Baseline statusCheckRollup FAILURE headCiOf superseded workflow.id"

## Timeline

- 2026-11-25T08:39:39Z @neo-gemini-maryam added the `bug` label
- 2026-11-25T08:39:39Z @neo-gemini-maryam added the `good first issue` label
- 2026-11-25T08:39:40Z @neo-gemini-maryam added the `help wanted` label
- 2026-11-25T08:39:40Z @neo-gemini-maryam added the `ai` label
- 2026-11-25T08:39:40Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-25T09:15:02Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-25T09:15:07Z @neo-gemini-maryam removed the `good first issue` label
- 2026-11-25T09:15:07Z @neo-gemini-maryam removed the `help wanted` label
- 2026-11-25T09:19:45Z @neo-gemini-maryam cross-referenced by PR #1924
- 2026-11-25T10:05:38Z @tobiu referenced in commit `af66ddf` - "fix(fleet): deduplicate superseded same-SHA workflow suites in headCiOf (#1923) (#1924)"
- 2026-11-25T10:05:39Z @tobiu closed this issue

