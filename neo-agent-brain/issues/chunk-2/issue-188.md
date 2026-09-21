---
id: 188
title: Retire the copied nightly E2E scheduler from Brain
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
  - testing
  - agent-os
assignees:
  - tobiu
createdAt: '2026-08-27T14:48:16Z'
updatedAt: '2026-08-27T14:55:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/188'
author: tobiu
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
closedAt: '2026-08-27T14:55:48Z'
---
# Retire the copied nightly E2E scheduler from Brain

## Context

Engine issue neomjs/neo#14685 introduced a host-only macOS LaunchAgent that ran the Engine whitebox-E2E config nightly and sent a red-only A2A digest. Brain PR neomjs/neo-agent-brain#178 copied the complete six-file cluster into Brain, while Engine PR neomjs/neo#17806 removed the same six files from Engine.

The source migration registry already classified `ai/scripts/lifecycle/nightly-e2e/com.neomjs.nightly-e2e.plist` as `stays-engine`, explicitly because the E2E plane was Engine-owned. The receive therefore contradicted its own custody input.

## The Problem

Brain cannot execute this scheduler as received:

- Brain contains zero `test/playwright/e2e/**` specs.
- Brain does not contain `test/playwright/playwright.config.e2e.mjs`.
- `nightlyE2eRunner.mjs` hard-codes that absent config and its Engine reporter output.
- No Brain npm script, workflow, daemon, or runtime module invokes the runner. Its only entrypoint is the staged plist.
- The inspected canonical workstation has neither an installed plist nor a loaded `com.neomjs.nightly-e2e` service.

The result is 1,434 lines of dormant scheduler, formatter, activation documentation, plist, and unit tests in a repository with no E2E suite to schedule. Open issues #14 and #17 currently try to make this non-capability observable instead of removing it.

## The Architectural Reality

The files are neither Brain runtime nor E2E coverage. They are an operator-side scheduler around a repository-local Playwright config. Browser E2E specs belong with the application repository they exercise; Brain should not carry a scheduler whose sole declared consumer is an absent Engine config.

The exact provenance is mechanically symmetric:

- Brain PR neomjs/neo-agent-brain#178 added the six nightly-E2E files.
- Engine PR neomjs/neo#17806 removed those same six paths.
- The migration registry said the plist stayed with Engine, not Brain.

This ticket corrects the receive error. It does not replace the scheduler elsewhere because the operator has explicitly retired it in both repositories.

## The Fix

1. Delete the runner, digest formatter, plist, activation README, and their two unit suites from Brain.
2. Remove only the live guard/test anchors that require the deleted runner: its permanent plane-literal acquittal and the Neural Link invariant's filesystem read of the runner as a regex positive control.
3. Preserve the actual Engine E2E suites and configs. They are not part of this cluster.
4. Leave the broader extraction-inventory retirement to its own authority; historical migration evidence may mention these paths, but no executable or live guard may depend on them.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Brain nightly-E2E LaunchAgent | neomjs/neo#14685 and operator retirement ruling | Removed with its runner and formatter | No replacement; Brain has no E2E suite | Delete activation README | Path and caller census |
| Engine whitebox-E2E suites | Engine `test/playwright/e2e/**` and `playwright.config.e2e.mjs` | Unchanged | Repository-local execution remains available | Existing Engine testing docs | Engine `dev` tree |
| Plane-literal guard | `buildScripts/util/check-aiconfig-antipatterns.mjs` | Remove only the deleted runner's exact-site acquittal | Remaining acquittals and their ratchet coverage stay intact | In-source rationale | Focused unit spec |
| Neural Link listener invariant | `relocationInvariants.spec.mjs` | Keep a synthetic positive control without reading the deleted runner | Neural Link zero-offender assertion stays unchanged | Test comments | Focused unit spec |

## Decision Record impact

`none` — this enforces the existing custody evidence and removes a dormant, non-executable receive artifact.

## Acceptance Criteria

- [ ] The six nightly-E2E cluster files added by Brain PR neomjs/neo-agent-brain#178 are absent from Brain.
- [ ] Engine `dev` remains unchanged and contains none of those six files after neomjs/neo#17806.
- [ ] No Brain npm script, workflow, import, or live guard references the retired runner, digest, plist, or state path.
- [ ] No Brain E2E config or E2E spec is introduced as a replacement.
- [ ] The AiConfig anti-pattern guard retains coverage for every surviving plane-literal acquittal.
- [ ] The Neural Link listener invariant retains a positive regex control without depending on the deleted runner.
- [ ] Focused affected unit specs pass at the exact PR head.
- [ ] After merge, #14 and #17 are reconciled as superseded where their scope depends on this scheduler.

## Out of Scope

- Deleting or relocating actual Engine whitebox-E2E specs.
- Redesigning E2E CI, nightly scheduling, or cross-repository test aggregation.
- Refactoring the broader extraction-inventory apparatus.
- Adding a replacement scheduler to Brain, Engine, Skills, or Agent Institution.

## Avoided Traps

- **Move it back to Engine.** Rejected: the operator retired the scheduler in both repositories, and Engine already removed it.
- **Make Brain able to run Engine E2E.** Rejected: that creates the cross-repository ownership error this cleanup removes.
- **Delete actual E2E coverage.** Rejected: only the scheduler cluster and its own unit coverage are in scope.
- **Fold the whole migration-inventory cleanup into this PR.** Rejected: it would turn a closed deletion into another mega-PR.

## Related

Related: #14, #17  
Source implementation: neomjs/neo#14685  
Receive: neomjs/neo-agent-brain#178  
Engine removal: neomjs/neo#17806

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-08-27T14:47:48Z; searched all Brain issues, the synced issue/discussion corpus, Knowledge Base, and the latest 30 all-state A2A messages; no retirement ticket or competing claim exists.

Origin Session ID: dfe02848-07ad-4dcf-a34a-9829f9c3dd9e

Retrieval Hint: `query_raw_memories("nightly e2e runner LaunchAgent #14685 Brain receive #178")`  
Retrieval Hint: Brain PR #178 and Engine PR #17806 exact six-file symmetry


## Timeline

- 2026-08-27T14:48:17Z @tobiu assigned to @tobiu
- 2026-08-27T14:48:18Z @tobiu added the `bug` label
- 2026-08-27T14:48:18Z @tobiu added the `ai` label
- 2026-08-27T14:48:18Z @tobiu added the `refactoring` label
- 2026-08-27T14:48:19Z @tobiu added the `testing` label
- 2026-08-27T14:48:19Z @tobiu added the `agent-os` label
### @tobiu - 2026-08-27T14:55:47Z

Authorship correction: this ticket was accidentally created through the operator credential. The identical agent-authored authority is #190; implementation proceeds there.

- 2026-08-27T14:55:48Z @tobiu closed this issue
- 2026-08-27T15:01:36Z @neo-gpt-emmy cross-referenced by #191

