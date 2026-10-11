---
id: 676
title: Test the locked dependencies before CI resolves dev
state: CLOSED
labels:
  - bug
  - ai
  - build
  - testing
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-11T01:56:01Z'
updatedAt: '2026-10-11T03:44:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/676'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 603
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-11T03:44:37Z'
---
# Test the locked dependencies before CI resolves dev

## Context

[Ada's review of #675](https://github.com/neomjs/neo-agent-institution/pull/675#pullrequestreview-5481528352) found a real install gap: since #660, `SeatLaunchAdmission.mjs` imported `isLaunchAdmissionProofReason`, while the locked Brain `2445eb36` did not export it. The move to `6b7e7d5a` in #675 repairs that instance.

## The Problem

Both jobs in `.github/workflows/ci.yml` run `npm ci` and then `resolve-org-dev` before tests. They validate current upstream development dependencies while replacing the versions an ordinary locked install receives. CI can therefore stay green while the locked application cannot link.

Independent control: importing the named symbol from the exact old `launchAdmission.mjs` fails with `does not provide an export named 'isLaunchAdmissionProofReason'`. The merged dependency contains it; the existing Institution unit suite passes with that lock (1708 passed, 11 skipped). This is a module-linking failure, not a subjective design preference.

The first run of the new locked-dependency stage also found the same class on the Engine: [run 38103637172](https://github.com/neomjs/neo-agent-institution/actions/runs/38103637172/job/114364483538) fails to load `VesselProxyEmbodiment.mjs` at locked `d75cc685`. The current FM consumer arrived in #670; its Engine provider arrived in neomjs/neo#19564 (`1c43d51e`). The repair therefore includes moving the Engine lock to merged `a852b8ca6e886eb32d583867259a24360f90a92c`, the existing candidate revision, and proving a fresh locked install. The prior local positive used newer package bytes than npm's recorded metadata and is not lock-faithful evidence.

## The Architectural Reality

`ci.yml` owns test ordering. `buildScripts/resolveOrgDev.mjs` deliberately moves the installed Brain and Engine to dev without changing the lock. `test/playwright/playwright.config.unit.mjs` already defines the isolated suite, and `unit/apps/agentos/util/seatLaunchAdmission.spec.mjs` imports the affected consumer.

Prescription checked: `.github/workflows/ci.yml` — owns the concern. Run the existing isolated unit suite immediately after `npm ci`, before any resolver/install replaces the locked dependencies. Retain the current dev-resolution and downstream checks. A new import scanner, script or duplicated dependency map is unnecessary.

## The Fix

Update the Engine lock to the named merged revision containing the already-consumed module. Add a clearly named locked-dependency unit step to the existing isolated job, before `resolve-org-dev`. Its ordinary failure must fail the job. Keep the post-resolution unit/component/E2E checks and the explicit Brain contract job.

| Surface | Authority | Behavior | Failure | Documentation | Evidence |
| --- | --- | --- | --- | --- | --- |
| Institution CI isolated job | Committed package lock, then existing dev resolver | Test the lock before replacing dependencies; retain dev testing | A locked import or unit failure fails CI | Workflow step name/comment | Workflow order and exact-head Actions run |

## Acceptance Criteria

- [ ] The isolated job runs the existing unit command after `npm ci` and before `resolve-org-dev`, without an intervening dependency update.
- [ ] A missing locked export/module fails that stage; retain the historical Brain control and the actual Engine CI failure, then prove the repaired Engine/Brain lock with a fresh install.
- [ ] Existing dev-resolution unit/component/E2E and explicit Brain contract coverage remain, and exact-head CI passes.

## Out of Scope

Dependency-security remediation (#657), unrelated dependency upgrades, changing dev tracking, a new import scanner, workflow matrices, installed FM acceptance or repository settings.

Decision Record impact: none. Structure map: N/A; Institution has no ai tree; the existing CI workflow and unit command own this change, with no new module.

Related: #675 · #603 · #660. This is the durable CI follow-up from the merged review; it does not reopen the delivered chooser.

Creation sweeps: latest 20 open Institution issues and latest 30 all-state A2A entries checked immediately before filing (2026-10-11 01:56Z); no equivalent claim or issue. Exact lock/resolve-org-dev searches found broader outcomes and #657, whose body owns advisory remediation rather than CI coverage. Own open Institution assignments: none. KB/MC problem queries returned onboarding/build prior art, no matching decision; live source and the negative control establish this gap. Six-stage check: reproduced defect, CI-owned ordering, existing command, maintainers/CI consumers, no new service boundary, no ADR change.

Origin Session ID: 3b93f03c-7a85-47e2-9dc7-3ba16a294c2b
Retrieval Hint: "Institution npm ci resolve-org-dev lock missing export SeatLaunchAdmission"


## Timeline

- 2026-10-11T01:56:01Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-11T01:56:03Z @neo-gpt-sophie added the `bug` label
- 2026-10-11T01:56:03Z @neo-gpt-sophie added the `ai` label
- 2026-10-11T01:56:03Z @neo-gpt-sophie added the `build` label
- 2026-10-11T01:56:04Z @neo-gpt-sophie added the `testing` label
- 2026-10-11T01:56:32Z @neo-gpt-sophie added parent issue #603
- 2026-10-11T01:57:20Z @neo-gpt-sophie cross-referenced by PR #675
- 2026-10-11T01:58:54Z @neo-gpt-sophie cross-referenced by PR #677
- 2026-10-11T02:11:21Z @neo-gpt-sophie referenced in commit `60f03c5` - "fix(build): lock the Engine required by the shipped vessel consumer (#676)"
- 2026-10-11T03:40:17Z @neo-gpt-sophie cross-referenced by PR #678
- 2026-10-11T03:44:37Z @tobiu referenced in commit `6c8ee99` - "fix(build): verify and repair locked org dependencies (#676) (#677)

* ci: test the locked dependencies before resolving dev (#676)

* fix(build): lock the Engine required by the shipped vessel consumer (#676)"
- 2026-10-11T03:44:37Z @tobiu closed this issue

