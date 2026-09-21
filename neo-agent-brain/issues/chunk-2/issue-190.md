---
id: 190
title: Retire the copied nightly E2E scheduler from Brain
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
  - testing
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-08-27T14:55:28Z'
updatedAt: '2026-08-28T07:30:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/190'
author: neo-gpt
commentsCount: 0
parentIssue: 191
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
closedAt: '2026-08-28T07:30:17Z'
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
2. Remove Brain's live test uses of the runner. Brain currently consumes the guard implementation from an Engine pin that predates neomjs/neo#17806; until that pin advances, keep one explicit negative compatibility assertion that the stale external allowlist row points to an absent Brain file.
3. Preserve the actual Engine E2E suites and configs. They are not part of this cluster.
4. Leave the broader extraction-inventory retirement to its own authority; historical migration evidence may mention these paths, but no executable or live guard may depend on them.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Brain nightly-E2E LaunchAgent | neomjs/neo#14685 and operator retirement ruling | Removed with its runner and formatter | No replacement; Brain has no E2E suite | Delete activation README | Path and caller census |
| Engine whitebox-E2E suites | Engine `test/playwright/e2e/**` and `playwright.config.e2e.mjs` | Unchanged | Repository-local execution remains available | Existing Engine testing docs | Engine `dev` tree |
| Plane-literal guard | Engine-pinned `buildScripts/util/check-aiconfig-antipatterns.mjs` | Brain no longer uses the runner as an active acquittal fixture; a temporary negative assertion quarantines the old pin's stale row | Newer Engine pin removes the external row | Brain consumer-spec comment | Focused unit spec |
| Neural Link listener invariant | `relocationInvariants.spec.mjs` | Keep a synthetic positive control without reading the deleted runner | Neural Link zero-offender assertion stays unchanged | Test comments | Focused unit spec |

## Decision Record impact

`none` — this enforces the existing custody evidence and removes a dormant, non-executable receive artifact.

## Acceptance Criteria

- [ ] The six nightly-E2E cluster files added by Brain PR neomjs/neo-agent-brain#178 are absent from Brain.
- [ ] Engine `dev` remains unchanged and contains none of those six files after neomjs/neo#17806.
- [ ] No Brain-owned npm script, workflow, runtime import, or executable source file references the retired runner, digest, plist, or state path.
- [ ] No Brain E2E config or E2E spec is introduced as a replacement.
- [ ] The AiConfig anti-pattern guard retains coverage for every surviving plane-literal acquittal; the pre-cut Engine pin's retired row is asserted only as a missing-target compatibility case.
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

- 2026-08-27T14:55:28Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T14:55:31Z @neo-gpt added the `bug` label
- 2026-08-27T14:55:31Z @neo-gpt added the `ai` label
- 2026-08-27T14:55:31Z @neo-gpt added the `refactoring` label
- 2026-08-27T14:55:32Z @neo-gpt added the `testing` label
- 2026-08-27T14:55:32Z @neo-gpt added the `agent-os` label
- 2026-08-27T14:55:48Z @tobiu cross-referenced by #188
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-27T15:16:03Z @neo-gpt cross-referenced by PR #203
- 2026-08-27T15:38:20Z @tobiu referenced in commit `6f56ebb` - "test(neural-link): strengthen listener red control (#190)"
- 2026-08-28T07:30:17Z @tobiu referenced in commit `c7be03e` - "Merge pull request #203 from neomjs/codex/190-retire-nightly-e2e

fix(lifecycle): retire copied nightly E2E scheduler (#190)"
- 2026-08-28T07:30:17Z @tobiu closed this issue
- 2026-08-28T22:23:25Z @neo-gpt-emmy cross-referenced by #14
- 2026-08-28T22:23:25Z @neo-gpt-emmy cross-referenced by #17
- 2026-08-30T19:48:15Z @neo-gpt-emmy cross-referenced by PR #259
- 2026-08-31T00:18:29Z @neo-gpt-emmy cross-referenced by #191
- 2026-09-06T21:17:48Z @neo-opus-vega cross-referenced by #17853

