---
id: 267
title: Correct devindex tenant seed to its live branch
state: CLOSED
labels:
  - bug
  - ai
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-31T00:03:57Z'
updatedAt: '2026-08-31T01:46:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/267'
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
closedAt: '2026-08-31T01:46:45Z'
---
# Correct devindex tenant seed to its live branch

## Context

The closed #12 Brain-image rehearsal reached the live pull-mode tenant seed and produced a terminal result for `neo-shared/devindex`: `KB_INGEST_ENVELOPE_REF_NOT_FOUND`. The repository was reachable; the configured ref was not.

Fresh remote evidence on 2026-08-31:

- `GET /repos/neomjs/devindex` reports `default_branch: dev`;
- the complete branch listing contains exactly `dev`;
- `deploy/cloud/kb-config.yaml` configures `branchRef: main` and claims every seeded remote was verified as main.

This is a tracked bootstrap defect, independent of image rehearsal and extraction-profile design.

## The Problem

`TenantRepoSyncService` correctly treats a missing configured head as terminal: elapsed time and retry cannot create a branch. The terminal-stop fingerprint is keyed by the configured ref, so the lane remains stopped while the seed keeps naming `main`.

The seed's explanatory prose and `KbTenantBootstrapContract.spec.mjs` both certify the same false value. Changing only the YAML without correcting those consumers would leave a green test and a future maintainer-facing claim that reintroduces the defect.

## The Architectural Reality

- `deploy/cloud/kb-config.yaml` is the Tier-2 bootstrap consumed by KB and Orchestrator.
- `TenantRepoSyncService` forwards `repo.branchRef || 'HEAD'` to the envelope builder.
- `isStoppedForCurrentInput()` suppresses only a matching `{ref, sourceErrorCode}` fingerprint. Changing `main` to `dev` naturally invalidates the old stop on the next sweep; no state-file surgery is needed.
- The deployment policy requires every seed entry to declare its own ref explicitly. Omitting `branchRef` would resolve the remote default today but weaken that audited policy.

Structure-map gate: `npm run --silent ai:structure-map -- --files --loc` passed. The owning surfaces are the declarative seed under `deploy/cloud/` and its existing contract spec at `test/playwright/unit/ai/deploy/KbTenantBootstrapContract.spec.mjs`.

## The Fix

1. Change only the `devindex` seed entry to `branchRef: dev`.
2. Correct the bootstrap comment: three seeded repositories remain `main`; `devindex` is the mixed-ref witness and follows its own live default branch, `dev`.
3. Update `KbTenantBootstrapContract.spec.mjs` so the other three refs remain pinned to `main` and `devindex` is pinned to `dev`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `tenants.neo-shared.tenantRepos[devindex].branchRef` | live `neomjs/devindex` remote refs | declare `dev` explicitly | omitting the field would use `HEAD` but weaken the per-repo explicit-ref policy; changing siblings would be unrelated | corrected inline bootstrap explanation | GitHub repository + branch readback; focused bootstrap contract spec |
| terminal-stop input | existing `isStoppedForCurrentInput` ref fingerprint | next sweep sees `dev` ≠ retained `main` and resumes normally | no manual checkpoint deletion or retry override | none | existing terminal-stop specs; no runtime code change |

## Decision Record impact

`none` — corrects one deployment coordinate without changing config precedence, sync semantics, or repository topology.

## Acceptance Criteria

- [ ] The canonical seed declares `branchRef: dev` for `neo-shared/devindex`.
- [ ] `create-app`, `devindex-opt-in`, and `devindex-opt-out` remain explicitly pinned to `main`.
- [ ] Bootstrap prose no longer claims every remote is main-only and explains the per-repo split accurately.
- [ ] `KbTenantBootstrapContract.spec.mjs` reds on `devindex: main` and passes on `devindex: dev`.
- [ ] The focused bootstrap contract suite passes.
- [ ] No tenant sync, content ingestion, checkpoint deletion, container, volume, KB, or MC mutation is mixed into this PR.

## Out of Scope

- Adding post-split tenant repositories.
- Implementing extraction profiles or parser-backed routes.
- Running the corrected sync.
- Editing terminal-stop or retry logic.
- Changing the other three seed refs.

## Avoided Traps

- **Omit `branchRef`.** Rejected: it hides the current defect behind remote default resolution and weakens the seed's explicit per-repo policy.
- **Clear persisted terminal state manually.** Rejected: the fingerprint already invalidates on an input-ref change.
- **Move every repo to `dev`.** Rejected: refs are per repository; the other three remotes remain main.
- **Fold into `#262`.** Rejected: this is a concrete deployment-coordinate bug, while `#262` owns the profile/materialization contract.

## Related

Related: #12, #262, #184, #253.

Origin Session ID: 4426fb43-4968-4084-832e-1830de2e8747

Retrieval Hint: `devindex branchRef main dev KB_INGEST_ENVELOPE_REF_NOT_FOUND canonical kb-config seed`

Live latest-open sweep: checked the latest 20 open Brain issues immediately before creation; no equivalent found.
A2A in-flight claim sweep: checked the latest 30 messages across all read states; Vega's measured lane input exists, but no competing claim or ticket intent.


## Timeline

- 2026-08-31T00:03:58Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-31T00:04:00Z @neo-gpt-emmy added the `bug` label
- 2026-08-31T00:04:00Z @neo-gpt-emmy added the `ai` label
- 2026-08-31T00:04:00Z @neo-gpt-emmy added the `build` label
- 2026-08-31T00:04:00Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-31T00:09:36Z @neo-gpt-emmy cross-referenced by PR #268
- 2026-08-31T00:10:47Z @neo-gpt-emmy cross-referenced by #262
- 2026-08-31T01:46:45Z @tobiu referenced in commit `dec8505` - "Merge pull request #268 from neomjs/codex/267-devindex-branch

fix(deploy): point devindex seed at live branch (#267)"
- 2026-08-31T01:46:45Z @tobiu closed this issue

