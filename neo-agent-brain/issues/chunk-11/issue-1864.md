---
id: 1864
title: Ignore informative pull_request_review checks in open-work CI state
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - performance
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-19T18:11:48Z'
updatedAt: '2026-11-19T21:26:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1864'
author: neo-gemini-julia
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
# Ignore informative pull_request_review checks in open-work CI state

Please comment to claim before starting; check current assignees and linked PRs.

## Context

On `2026-11-19`, `@tobiu` reported `5+ minute` delays between a maintainer submitting an `APPROVED` review on GitHub and Fleet Manager displaying the PR as merge-ready (`awaitingMerge`). Measuring pulse transitions across approved Engine PRs on `2026-11-19` isolated a `3–4+ minute` blind spot triggered by the post-submit review-policy workflow:

- `neomjs/neo#21485`: `17:17:26Z` `verdict: CHANGES_REQUESTED -> APPROVED` coincided with `ci: green -> pending`, staying `pending` until `17:20:26Z` (`3m 00s`).
- `neomjs/neo#21486`: `17:31:26Z` `verdict: CHANGES_REQUESTED -> APPROVED` coincided with `ci: green -> pending`, staying `pending` until `17:35:26Z` (`4m 00s`).
- `neomjs/neo#21487`: `17:36:13Z` `APPROVED` review submitted -> `17:36:26Z` `ci: green -> pending`, staying `pending` until `17:40:26Z` (`4m 13s`).

## The Problem

Submitting or editing a GitHub review triggers `neomjs/neo`'s `.github/workflows/pr-review.yml` (`neomjs/neo-agent-skills` `.github/workflows/reusable-pr-review.yml`) on `pull_request_review: [submitted, edited]`. Under the `20`-runner org concurrency ceiling, that workflow queues for `3–4+ minutes` behind sharded Playwright runs.

While that `pull_request_review` check suite is queued or in progress, GitHub's coarse `commit.statusCheckRollup.state` flips from `SUCCESS` to `PENDING` on an already-green head SHA (and to `FAILURE` if an informative review-format lint fails on a `COMMENT` review). `normalizePullRequest` in `ai/services/fleet/openWorkReducer.mjs` maps `commit.statusCheckRollup.state` directly into `row.ci`, so `holderOf` in `ai/services/fleet/openWorkHolder.mjs` (`if (ci === 'green' && approved && mergeable === 'MERGEABLE' && !draft)`) drops the PR to `{role: 'none', kind: null}` instead of `{role: 'operator', kind: 'merge'}` (`awaitingMerge`) for the entire queue + run duration of the review lint.

## The Architectural Reality

- `Design authority: "the check is informative rather than required: it runs once per review, a new head carries no result until the next review, and requiring it would hold the merge on that gap."` (`neomjs/neo-agent-skills` `.github/workflows/reusable-pr-review.yml`, lines 6–7; mirrored in `neomjs/neo` `.github/workflows/pr-review.yml`, lines 2–3).
- `ai/services/fleet/openWorkReducer.mjs` (line 172) already excludes `pull_request_review` suites when computing `runApproval` (`suite.workflowRun?.event !== 'pull_request_review'`), but line 182 still reads `CI_STATES[commit?.statusCheckRollup?.state] ?? null` unfiltered.
- `ai/services/github-workflow/queries/openWorkQueries.mjs` (lines 251–261) already fetches `commit.checkSuites(last: 100) { pageInfo { hasPreviousPage } nodes { conclusion workflowRun { event } } }` on every `OPEN_WORK_SNAPSHOT` and `OPEN_WORK_OUTSIDE_SNAPSHOT` page. Adding scalar `status` on `checkSuites.nodes` plus `status { state contexts { state } }` (legacy commit statuses such as `review-admission/mergeability`) keeps `OpenWorkSnapshot` at `rateLimit.cost === 1` (verified via `gh api graphql`).
- `ai/services/fleet/openWorkHolder.mjs` (lines 233–241): requires `ci === 'green'` (or `ci === null && mergeState === 'CLEAN'`) to assign `awaitingMerge`.

## The Fix

1. **`ai/services/github-workflow/queries/openWorkQueries.mjs` (`snapshotQuery`):**
   - Select `status` beside `conclusion` on `commit.checkSuites(last: 100).nodes`.
   - Select `status { state contexts { state } }` on `commit` beside `statusCheckRollup { state ... }` so legacy `StatusContext` state (`PENDING` / `FAILURE` / `ERROR` / `SUCCESS`) is distinguishable from `CheckSuite` state without paginating `statusCheckRollup.contexts` on lean seat reads.
2. **`ai/services/fleet/openWorkReducer.mjs` (`normalizePullRequest`):**
   - Introduce a pure helper deriving effective head `ci` when `suites.complete` is `true` and at least one non-`pull_request_review` check suite or commit `status` context exists: ignore `checkSuites.nodes` whose `workflowRun?.event === 'pull_request_review'` when aggregating head CI state (`red` if any remaining checkSuite has failing conclusion or `commit.status.state` is `FAILURE`/`ERROR`; `pending` if any remaining checkSuite has `status !== 'COMPLETED'` or `commit.status.state` is `PENDING`/`EXPECTED`; `green` when at least one head checkSuite or status context completed cleanly and none are failing/pending).
   - Preserve exact fallback to `CI_STATES[commit?.statusCheckRollup?.state] ?? null` whenever `suites` is incomplete (`!suites.complete`) or when legacy test fixtures omit `checkSuites` / `status` details and provide only `statusCheckRollup.state`.
3. **Unit tests (`test/playwright/unit/ai/services/fleet/openWorkReducer.spec.mjs`):**
   - Add red-first unit specs proving: (a) an approved head whose head `pull_request` / `pull_request_target` check suites and `review-admission/mergeability` `StatusContext` are `COMPLETED`/`SUCCESS` stays `ci: 'green'` (and `holderOf` returns `{role: 'operator', kind: 'merge'}`) while a `pull_request_review` suite is `QUEUED` or `IN_PROGRESS` with `statusCheckRollup.state === 'PENDING'`; (b) a pending `StatusContext` or pending `pull_request` check suite still yields `ci: 'pending'`; (c) an incomplete `checkSuites` page (`hasPreviousPage: true`) safely falls back to `statusCheckRollup.state`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge | Docs | Evidence |
|---|---|---|---|---|---|
| `snapshotQuery` (`ai/services/github-workflow/queries/openWorkQueries.mjs`) | `reusable-pr-review.yml` informative contract; this ticket | Fetch `checkSuites.nodes.status`, `checkSuites.nodes.workflowRun.workflow.id`, and `commit.status { state contexts { state } }` at unchanged query cost (`5` on `OPEN_WORK_SNAPSHOT`, `6` on `OPEN_WORK_OUTSIDE_SNAPSHOT`) | `commit.status === null` when a repo uses only Actions check suites; retained previous query hashes stay priced in `OPEN_WORK_QUERY_COSTS` | JSDoc on `snapshotQuery` | `openWorkQueryCost.spec.mjs` / `openWorkReducer.spec.mjs` |
| `normalizePullRequest` (`ai/services/fleet/openWorkReducer.mjs`) | `openWorkReducer.mjs:172` + `reusable-pr-review.yml` | Exclude `workflowRun.event === 'pull_request_review'` check suites from head `ci` derivation when `suites.complete`, keeping the newest suite per `workflow.id` (so a superseded run of the same workflow on the same head is ignored) while short-circuiting when `statusCheckRollup` is already `green` | Fall back to `CI_STATES[statusCheckRollup.state]` when `!suites.complete`, no `pull_request_review` suite is present, or fixture omits `commit.status` / suite `status` | Anchor & Echo JSDoc | `openWorkReducer.spec.mjs` |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1: `snapshotQuery` in `ai/services/github-workflow/queries/openWorkQueries.mjs` selects `checkSuites.nodes.status` and `commit.status { state contexts { state } }` without increasing single-page GraphQL query cost (`openWorkQueryCost` / query specs updated if needed).
- [ ] AC-2: `normalizePullRequest` in `ai/services/fleet/openWorkReducer.mjs` keeps `ci === 'green'` when head `checkSuites` (`event !== 'pull_request_review'`) and `commit.status` have succeeded even while a `pull_request_review` suite is queued/in-progress (`statusCheckRollup.state === 'PENDING'`) or failed (`statusCheckRollup.state === 'FAILURE'`).
- [ ] AC-3: `normalizePullRequest` still reports `ci === 'pending'` when any non-`pull_request_review` check suite is non-`COMPLETED` or `commit.status.state` is `PENDING`/`EXPECTED`, and falls back to `statusCheckRollup.state` when `!suites.complete`.
- [ ] AC-4: Unit specs in `test/playwright/unit/ai/services/fleet/openWorkReducer.spec.mjs` cover AC-2 and AC-3 (verified via `npm test`).

## Out of Scope

- Replacing cockpit and Electron shell polling with event-driven SSE push (tracked in sibling Brain + Institution push tickets).
- Aggregating or trimming Engine GitHub Actions runner jobs (`neomjs/neo#21500`, `neomjs/neo-agent-skills#427`).

## Avoided Traps

- **Blindly treating `statusCheckRollup.state === 'PENDING'` as `'green'` whenever `verdict === 'APPROVED'`:** unsafe — if a base-branch update flips `review-admission/mergeability` (`StatusContext`) or a re-run check suite to `PENDING`, the head genuinely has pending CI and must not enter `awaitingMerge`. Inspecting `commit.status` + non-`pull_request_review` `checkSuites` distinguishes the two cleanly.
- **Paginating `statusCheckRollup.contexts(first: 100)` on every seat `OPEN_WORK_SNAPSHOT` search:** inflates GraphQL node complexity across `50` PRs × `100` contexts (`5,000` nodes/page), whereas `commit.status` + already-fetched `checkSuites(last: 100)` costs `1` point.

## Related

- #1842 (parallelized seat GraphQL searches and bounded `readOwnQuestions` timeout)
- #1221 (repos without CI)
- `neomjs/neo#21500` and `neomjs/neo-agent-skills#427` (org `20`-runner CI budget)

---

Live latest-open + closed sweep: checked latest 20 open of 160 (`red` band; `#1863`..`#1719`) + open and closed searches (`fleetOpenWork`, `statusCheckRollup`, `review-lint`, `openWorkHolder`, `#1842`, `#1221`, Out of Scope / Avoided Traps read on `#1842` and `#1689`) at `2026-11-19T18:10Z`; no equivalent or prior exclusion found.
A2A in-flight claim sweep: latest 30 messages across all read-states (`17:32Z–18:09Z`); no overlapping claim (`MESSAGE:a03c0a54` covers runner-job count in `neo#21500` / `skills#427`, not open-work `ci` normalization).
MC sweep: `"Fleet Manager UI laggy slow approved PRs minutes awaitingMerge statusCheckRollup pending"`, 8 results, no prior decision found.
Own-assignment sweep: 1 open (`#1230`), none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): owning files `ai/services/fleet/openWorkReducer.mjs` and `ai/services/github-workflow/queries/openWorkQueries.mjs`; no new files.

unowned-rationale: Ready for a contributor or peer maintainer; maintainers provide planning and review.

Origin Session ID: a4bf6af6-691f-41a3-a060-2f8ab8b4ff81
Retrieval Hint: "approved PR disappears awaitingMerge 4 minutes pull_request_review statusCheckRollup pending openWorkReducer"

## Timeline

- 2026-11-19T18:11:49Z @neo-gemini-julia added the `bug` label
- 2026-11-19T18:11:50Z @neo-gemini-julia added the `good first issue` label
- 2026-11-19T18:11:50Z @neo-gemini-julia added the `help wanted` label
- 2026-11-19T18:11:50Z @neo-gemini-julia added the `ai` label
- 2026-11-19T18:11:50Z @neo-gemini-julia added the `performance` label
- 2026-11-19T18:11:51Z @neo-gemini-julia added the `agent-os` label
- 2026-11-19T20:31:26Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-19T20:38:21Z @neo-gemini-julia cross-referenced by PR #1867
- 2026-11-19T21:27:11Z @neo-gemini-julia referenced in commit `98e04ba` - "fix(fleet): document headCiOf short-circuit and checkSuites order (#1864)"

