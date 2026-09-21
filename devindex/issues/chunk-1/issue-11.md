---
id: 11
title: Adopt the shared PR baseline in DevIndex
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-29T12:53:14Z'
updatedAt: '2026-08-29T16:15:01Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/11'
author: neo-gpt-emmy
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
closedAt: '2026-08-29T16:15:01Z'
---
# Adopt the shared PR baseline in DevIndex

## Context

`neomjs/neo-agent-skills` now owns the reusable PR baseline at merged commit `72965ba56b43e898d566af97f8ab277b28beb96b` (Skills PR #17, leaf #15 under Epic #14). DevIndex already installs `neo-agent-skills` and runs `neo-agent-skills-materialize` from postinstall, but its current `dev` tree has no materialization check and no shared PR-base guard.

The first consumer must prove the cross-repository call rather than copying the source workflow. DevIndex is the cleanest witness because there is no existing `substrate-sync.yml` to replace and no overlap with the Engine/Brain enforcement-settings lane in `neomjs/neo#17783`.

Structure-map gate: the Brain structure map was run for Epic #14. This leaf adds no Agent OS runtime source; its only placement precedent is DevIndex's existing `.github/workflows/ci.yml` caller surface.

## The Problem

DevIndex consumes the canonical Skills package but has no PR check proving the projection happened. Adding another local implementation would recreate the duplication Epic #14 exists to remove. A caller scoped only to `pull_request.branches: [dev]` would also create a branch-protection trap: once the shared job names become required, a release PR targeting another branch would never emit them and would remain pending rather than fail clearly.

## The Architectural Reality

- The reusable implementation lives in `neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml`.
- Cross-repository reusable workflows are invoked at the job level with `uses:` and execute against the caller repository's event/context.
- The immutable merged Skills commit is the consumer pin; mutable `dev` is not an acceptable required-CI coordinate.
- DevIndex owns only the event trigger, least-privileged caller permissions, and the pinned `uses:` line. Its product build/unit/e2e workflow remains local.
- The caller must trigger on every PR. The reusable `PR base` job decides whether the target branch is permitted and emits a terminal result for wrong-base PRs.

## The Fix

Add one thin DevIndex caller workflow:

- `on: pull_request` with no branch filter;
- top-level `contents: read` only;
- one stable caller job invoking the merged Skills reusable workflow at commit `72965ba56b43e898d566af97f8ab277b28beb96b`;
- no copied install, base-check, or materializer steps.

Use the PR that introduces the caller as the live integration witness: both called jobs must appear and pass against the DevIndex checkout. Record their exact emitted status names for the later required-context binding lane.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| DevIndex PR-baseline caller | Skills Epic #14 + merged PR #17 | Calls the canonical reusable workflow from one pinned job | Missing/inaccessible reusable source is red | Caller comments | Live PR check run |
| Workflow pin | Skills merge commit `72965ba56b43e898d566af97f8ab277b28beb96b` | Immutable source coordinate | No mutable branch fallback | Caller `uses:` line | GitHub resolves the call on this PR |
| PR trigger | Review-routing contract + required-context behavior | Runs on every pull request; reusable job decides allowed base | Wrong-base PR gets a terminal red `PR base`, never a missing/pending context | Caller comment | Workflow event plus negative contract inherited from Skills #15 |
| Product CI | DevIndex `.github/workflows/ci.yml` | Remains repository-owned and unchanged | Shared baseline failure does not rewrite product-test semantics | Existing workflow | Zero diff to product CI |

## Decision Record impact

`none` — consumer activation of the accepted Skills SSOT; no runtime or product ADR changes.

## Acceptance Criteria

- [ ] DevIndex carries one thin reusable-workflow caller pinned exactly to Skills commit `72965ba56b43e898d566af97f8ab277b28beb96b`.
- [ ] The caller triggers on all `pull_request` events with no branch filter.
- [ ] Caller permissions are limited to `contents: read`.
- [ ] The caller contains no copied checkout, Node, install, base-decision, or materializer implementation steps.
- [ ] The introducing PR emits both reusable jobs against the DevIndex checkout and both pass; the PR body records their exact status names.
- [ ] Existing DevIndex product CI and test sources are unchanged.

## Out of Scope

- Binding required contexts in repository settings (`neomjs/neo#17783` coordinates that settings class).
- PR-body/review-body/commit-authorship or `agent-preflight` migration.
- Engine, Brain, Institution, or Skills-source changes.
- Product build, unit, e2e, or data-sync behavior.

## Avoided Traps

- Copying `substrate-sync.yml` into DevIndex.
- Pinning the caller to mutable `dev`.
- Filtering the caller to `branches: [dev]` and creating permanently missing required checks on other PR bases.
- Adding postinstall logic that writes tracked workflow files.
- Treating product CI parity as part of non-product PR governance.

## Related

- `neomjs/neo-agent-skills#14` — shared PR-governance Epic.
- `neomjs/neo-agent-skills#15` / PR #17 — landed reusable source.
- `neomjs/neo#17783` — required-context/settings and guard-caller enforcement.

Live latest-open sweep: checked all 3 open DevIndex issues created-descending at 2026-08-29T12:53Z; no equivalent found. A2A in-flight claim sweep: latest 30 messages across all read states; no overlapping claim.

Origin Session ID: 5ea998d3-e1ed-4214-8472-c337ff247403

Retrieval Hint: `DevIndex thin caller reusable PR baseline Skills 72965ba all pull requests`

## Timeline

- 2026-08-29T12:53:16Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-29T12:53:16Z @neo-gpt-emmy added the `ai` label
- 2026-08-29T12:53:16Z @neo-gpt-emmy added the `architecture` label
- 2026-08-29T12:53:20Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T12:56:12Z @neo-gpt-emmy cross-referenced by PR #12
- 2026-08-29T16:15:01Z @tobiu referenced in commit `50da13b` - "Merge pull request #12 from neomjs/codex/11-shared-pr-baseline

feat(ci): adopt shared PR baseline (#11)"
- 2026-08-29T16:15:01Z @tobiu closed this issue
- 2026-08-30T21:12:53Z @neo-gpt-emmy cross-referenced by #64

