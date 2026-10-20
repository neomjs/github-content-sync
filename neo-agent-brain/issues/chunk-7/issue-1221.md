---
id: 1221
title: Approved PRs from a repository without CI never reach awaiting merge
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-20T09:26:21Z'
updatedAt: '2026-10-20T09:26:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1221'
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
# Approved PRs from a repository without CI never reach awaiting merge

## Context

The operator noticed on 2026-10-20 that the Fleet head's awaiting-merge list held four PRs (neo #19873, #19784, Institution #761, Brain #1216), while the activity stream below it showed neo-stage PRs being approved. His guess was that neo-stage is still missing from some Agent OS target list.

That guess does not hold. The open-work producer reads neo-stage on every pulse: the live `open-work.json` (observed 09:22:36Z) carries 7 neo-stage rows and 1 pages row. The rows are read and then dropped by the holder rule.

## The Problem

`holderOf()` (`ai/services/fleet/openWorkHolder.mjs`) gives the operator a PR's next action (`role: 'operator', kind: 'merge'`) only when `ci === 'green'`. The reducer sets `ci` from the head commit's `statusCheckRollup.state` and leaves it `null` when the head reports no checks at all (`openWorkReducer.mjs`, `CI_STATES[…] ?? null`). Repositories that run no CI report `null` permanently: neo-stage, whose hosted CI is paused by repo policy, and pages, which runs no workflows. So their approved, mergeable PRs fall through every `ci === 'green'` rule to `role: 'none'` and never enter `awaitingMerge`.

At 2026-10-20 ~09:25Z, seven such PRs were APPROVED with `mergeStateStatus: CLEAN` on live GitHub and absent from the list: neo-stage #221, #222, #223, #227, #228, #229 and pages #46. Each had `ci: null`, `verdict: APPROVED`, `mergeable: MERGEABLE`, `draft: false` in the snapshot.

## The Architectural Reality

- `projectOpenWork()` (`fleetOpenWorkSource.mjs`) pushes a row to `awaitingMerge` exactly when `holderOf(row).role === 'operator'`. The wake path reads the same function (`openWorkWakes.mjs`), so the two never disagree.
- `ci === null` alone cannot separate "this repository runs no CI" from "checks have not registered on a fresh head yet". GitHub's `mergeStateStatus` can: it is `CLEAN` only when nothing the repository requires still blocks the merge, required checks included. GraphQL search returns it without a preview header (verified on neo-stage #228/#229: `CLEAN`, rollup `null`, 0 check suites; neo #19881: `BLOCKED` with rollup `SUCCESS`).
- `CLEAN` alone is not readiness: neo-stage #230 is `CLEAN` while changes are requested, because neo-stage requires no review. The approval on the current head stays required.

## The Fix

1. `OPEN_WORK_SNAPSHOT` (`ai/services/github-workflow/queries/openWorkQueries.mjs`) also reads `mergeStateStatus`; `normalizePullRequest()` carries it as `mergeState` (`null` when absent).
2. `holderOf()` gains one rule after the existing green rules: a head that reports no CI (`ci === null`) with an approval on the head, `mergeable === 'MERGEABLE'`, not a draft and `mergeState === 'CLEAN'` is the operator's to merge. A change request on the head, a red or pending CI and every green path keep their current holders.
3. Unit arms beside the existing holder and reducer specs.

## Acceptance Criteria

- [ ] AC-1 A row with `ci: null`, an approval on the head, `MERGEABLE`, not a draft and `mergeState: 'CLEAN'` is held by `{role: 'operator', kind: 'merge'}` and reaches `awaitingMerge`.
- [ ] AC-2 The same row with `mergeState` `BLOCKED`, `UNSTABLE` or `null` keeps `role: 'none'`; with a change request on the head it stays the author's; a green row's holder is unchanged.
- [ ] AC-3 `normalizePullRequest()` maps a node's `mergeStateStatus` to `mergeState` and missing to `null`; the snapshot query names the field.
- [ ] AC-4 (post-merge, installed FM) After the next pulse, the Fleet head's awaiting-merge list shows any approved, `CLEAN` neo-stage or pages PR open at that time.

## Out of Scope

- The reviewer and route-review holders for no-CI repositories: they also require `ci === 'green'`, so a requested reviewer on neo-stage or pages gets no FM wake. Changing wake timing in repositories that do run CI needs its own decision; noted as a defect-note.
- Re-enabling neo-stage's hosted CI (repo policy).

## Related

#780 (the holder, mine) · #759 · neomjs/neo-stage#221 · neomjs/pages#46

Live latest-open sweep at 2026-10-20 09:26Z (latest 20 open Brain issues; searches "awaiting merge", "holderOf ci", open PRs touching `openWorkHolder`) and the last 30 A2A messages: no equivalent or in-flight claim. Memory Core sweep on the symptom: no prior decision about repositories without CI.

Origin Session ID: ad84dc14-8baf-4f95-a7dd-a9e224f51c31

Retrieval Hint: "awaiting merge drops approved PRs from repositories without CI"

## Timeline

- 2026-10-20T09:26:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-20T09:26:23Z @neo-opus-ada added the `bug` label
- 2026-10-20T09:26:23Z @neo-opus-ada added the `ai` label
- 2026-10-20T09:26:23Z @neo-opus-ada added the `agent-os` label
- 2026-10-20T09:33:18Z @neo-opus-ada cross-referenced by PR #1222

