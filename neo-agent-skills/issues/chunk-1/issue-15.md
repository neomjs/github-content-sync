---
id: 15
title: Publish the reusable PR-baseline workflow
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - testing
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T11:39:17Z'
updatedAt: '2026-08-29T12:50:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/15'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 14
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T12:50:23Z'
---
# Publish the reusable PR-baseline workflow

Parent Epic: #14

## Context

Engine and Brain currently carry separate `substrate-sync.yml` implementations whose executable behavior is the same, while DevIndex and Institution consume `neo-agent-skills` without the same current-tree check. Engine separately owns the PR base guard. The first canonical leaf needs to prove that Skills can own multi-job consumer CI before deeper PR-body and review-body policy moves.

Live evidence on 2026-08-29: every relevant consumer's `dev` package manifest installs `neo-agent-skills` and invokes `neo-agent-skills-materialize`; Engine and Brain are the only current trees carrying the materialization workflow; all five `dev` branches have zero required status contexts.

## The Problem

The same distribution invariant is implemented in consumer YAML, so every new repository must copy the logic or silently lack it. The old N=2 rationale for keeping copies no longer holds at N=5. Copying more files would create parity by convention, not by construction.

## The Architectural Reality

- GitHub reusable workflows live under `.github/workflows` and expose `on.workflow_call`.
- A cross-repository called workflow receives the caller's `github` context, and `actions/checkout` checks out the caller repository.
- The Skills npm package already exposes `neo-agent-skills-materialize --check`; the reusable workflow should invoke that supported consumer surface rather than duplicate linker logic.
- `.github/**` remains excluded from the npm tarball. GitHub consumes the reusable workflow from the Skills repository; npm carries the skill corpus and materializer.
- Structure-map placement: the Brain map was run; this leaf adds no Brain runtime source. The sibling precedent is `.github/workflows/skill-corpus.yml` in this repository.

## The Fix

Add one least-privileged reusable PR-baseline workflow with two independently named jobs:

1. A base-branch job that accepts a required base input defaulting to `dev` and refuses a caller PR targeting another branch.
2. A Skills-materialization job that checks out the caller, installs its lockfile with Node 24, and runs `npx --no-install neo-agent-skills-materialize --check`.

Add a canonical-repo contract test that fails if the reusable workflow loses `workflow_call`, broadens permissions, checks out the wrong repository, drops either job, or replaces the supported materializer command. Run that test from `skill-corpus.yml`. Keep workflow and test files out of the published tarball.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Reusable PR-baseline workflow | Epic #14 + GitHub `workflow_call` contract | Exposes stable base and Skills-materialization jobs to pinned cross-repo callers | Non-PR invocation fails the base contract rather than reporting a false success | Workflow comments | Contract test + GitHub workflow parser |
| Required base input | Agent PRs target `dev`; releases remain human-directed | Defaults to `dev`, caller may explicitly select another protected development branch | No implicit default-branch lookup | Workflow input description | Positive `dev` and negative alternate-base fixtures |
| Skills materialization job | `neo-agent-skills-materialize --check` | Installs the caller lockfile and proves projected skills are reachable and unshadowed | Missing dependency/postinstall projection is red | Existing materializer help | Existing materializer fixtures + workflow contract test |
| Published package | `package.json#files` + `skill-corpus.yml` package check | Continues shipping skills and materializer, not CI/test implementation | Tarball containing `.github` or contract-test source is red | Package manifest | Existing tarball negative checks |

## Decision Record impact

`none` — this completes the selected npm Skills SSOT and corrects the stale reusable-workflow exclusion; no runtime ADR changes.

## Acceptance Criteria

- [ ] A reusable workflow under `.github/workflows` declares `workflow_call` and only least-privileged read access.
- [ ] It exposes separately named base-branch and Skills-materialization jobs with stable names suitable for required status contexts.
- [ ] The base job defaults to `dev` and fails on a mismatched caller PR base.
- [ ] The materialization job checks out the caller, runs Node 24 + its lockfile install, and executes `neo-agent-skills-materialize --check`.
- [ ] A canonical-repo contract test covers the two positive contracts plus negative drift in trigger, permissions, checkout ownership, and materializer command; `skill-corpus.yml` executes it.
- [ ] `npm pack` still excludes `.github/**` and the workflow contract-test implementation.
- [ ] No consumer repository is modified by this PR; consumer callers remain separate Epic leaves after the reusable source lands.

## Out of Scope

- PR-body, review-body, review-cost, or commit-authorship policy migration.
- Consumer caller files and required-status settings.
- Product build, unit, integration, e2e, visual, or domain-specific lints.
- Writing workflow files from postinstall.

## Avoided Traps

- Copying Engine's workflow into three more repositories.
- Publishing `.github` inside the npm tarball even though GitHub, not Node resolution, consumes it.
- Referencing `dev` as a mutable cross-repository workflow coordinate in consumers.
- Hiding both checks behind one opaque shell step/status.
- Adding a registry, receipt file, or generated workflow inventory.

## Related

- Parent Epic #14.
- `neomjs/neo#17783` — required status and caller enforcement lane.
- `neomjs/neo#17784` — Skills distribution history.

Live latest-open sweep: checked all 10 open Skills issues created-descending at 2026-08-29T11:39Z; no equivalent found. A2A in-flight claim sweep: latest 30 messages across all read states; no overlapping claim.

Origin Session ID: f6ad5621-a6d8-4636-bcf6-5fdab898bed0

Retrieval Hint: `reusable PR baseline workflow base guard substrate materialization neo-agent-skills`

## Timeline

- 2026-08-29T11:39:19Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-29T11:39:19Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T11:39:19Z @neo-gpt-emmy added the `architecture` label
- 2026-08-29T11:39:19Z @neo-gpt-emmy added the `testing` label
- 2026-08-29T11:39:19Z @neo-gpt-emmy added the `build` label
- 2026-08-29T11:39:28Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T11:39:31Z @neo-gpt-emmy added parent issue #14
- 2026-08-29T11:46:29Z @neo-opus-ada cross-referenced by PR #16
- 2026-08-29T11:48:01Z @neo-gpt-emmy cross-referenced by PR #17
- 2026-08-29T12:37:36Z @tobiu referenced in commit `479ef67` - "fix(ci): cover aggregate workflow permissions (#15)"
- 2026-08-29T12:50:23Z @tobiu referenced in commit `72965ba` - "Merge pull request #17 from neomjs/codex/15-reusable-pr-baseline

feat(ci): publish reusable PR baseline (#15)"
- 2026-08-29T12:50:23Z @tobiu closed this issue
- 2026-08-30T00:44:19Z @neo-gpt cross-referenced by #18
- 2026-08-30T17:53:59Z @neo-gpt-emmy cross-referenced by #22
- 2026-08-30T18:03:13Z @neo-gpt-emmy cross-referenced by #14
- 2026-08-30T18:04:43Z @neo-gpt-emmy cross-referenced by #23

