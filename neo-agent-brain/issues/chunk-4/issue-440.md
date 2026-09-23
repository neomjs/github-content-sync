---
id: 440
title: Two lease-loss specs race their own renewal ticks for the lifecycle guard
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T14:39:39Z'
updatedAt: '2026-09-23T17:11:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/440'
author: neo-opus-vega
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
closedAt: '2026-09-23T17:11:37Z'
---
# Two lease-loss specs race their own renewal ticks for the lifecycle guard

## Context

Brain Unit fails a run that has any flaky test, and `TenantRepoSyncService.spec.mjs:6756` ("renewal failure aborts before protected work… (#15763)") flaked in three of the last six runs at `ef13cdb`: `dev` itself (run 35870037946, 13:51Z, `ENOENT … mkdir '…/tenant-repo-sync-lease.json.lifecycle-guard'`) and both attempts of PR #439's unit job (run 35874158366, `ENOENT … rmdir '…lifecycle-guard'`). PR #439 does not touch the file. Locally, `--repeat-each=40 --retries=0` on that test fails 38 of 40 with `ENOTEMPTY … rmdir '…lifecycle-guard'`. Every Brain PR's unit job inherits the coin flip.

## The Problem

The test simulates a reclaimer replacing the lease "INSIDE the lifecycle guard so an in-flight renewal tick cannot interleave with this test write", in its own words. But it enters the guard with `fs.ensureDir(guardPath)` and leaves with `fs.rmdir(guardPath)`, and `ensureDir` is not an exclusive acquire. When one of the run's 25 ms renewal ticks (`leaseRenewalIntervalMs: 25`) already holds the guard, `ensureDir` succeeds on the existing directory and the test writes while the tick is still inside. What follows depends on timing: the tick exits first and removes the directory (the test's `rmdir` gets `ENOENT`), the tick is still inside (`rmdir` gets `ENOTEMPTY`), or the directory disappears mid-create (`mkdir` gets `ENOENT`). The spec at `:7205` (the evicted predecessor's in-flight record) uses the same two lines.

## The Architectural Reality

- `ai/daemons/shared/lifecycleGuard.mjs`: `enterLifecycleGuard({leasePath, fsModule})` acquires exclusively (a staging `mkdir` plus rename, an owner token, bounded retry, and a stale steal) and returns `{ownerFilePath}`, or `null` on contention. `exitLifecycleGuard({ownerFilePath, fsModule})` releases it. Every production `rmdir` in the module already tolerates `ENOENT` / `ENOTEMPTY`.
- The spec imports both helpers already and uses them correctly at `:7456` and `:7571`.

## The Fix

At `:6781` and `:7205`, replace the `ensureDir` / `rmdir` pair with `enterLifecycleGuard` (asserting a non-null hold) and `exitLifecycleGuard` in a `finally`. The change is test-only.

## Acceptance Criteria

- [ ] **AC-1** Both specs enter and leave the guard through the helpers, and `--repeat-each=40 --retries=0` on each passes 40/40 locally (`:6756` fails 38/40 today).
- [ ] **AC-2** The full `TenantRepoSyncService.spec.mjs` stays green, and the PR's Brain Unit run reports no flaky result at either site.

## Out of Scope

Brain Unit's fail-on-flaky policy, which is right to fail here. Other flaky specs.

## Related

neomjs/neo#15763 and neomjs/neo#15772 (the guard these specs exercise) · #439 (red on it) · #201

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T14:39Z, none equivalent. Org keyword sweeps (`lifecycle-guard`, `renewal failure aborts before protected work`, `ENOENT rmdir`) return no ticket for this flake. A2A in-flight sweep: my own defect-note at 14:31Z is the only sighting, promoted by this ticket. MC sweep: `query_raw_memories` (5) returned the July work that introduced the guard (neomjs/neo#15772) and no prior decision on this flake. Own-assignment sweep: #438, #434, #432, #430, #417, #64, #65, #23, none overlapping. Structure map: N/A, test-only.

Origin Session ID: 603e5af2-9d35-4bfc-9852-038c4cf38568

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-23T14:39:40Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T14:39:41Z @neo-opus-vega added the `bug` label
- 2026-09-23T14:39:41Z @neo-opus-vega added the `ai` label
- 2026-09-23T14:39:41Z @neo-opus-vega added the `testing` label
- 2026-09-23T14:42:34Z @neo-opus-vega cross-referenced by PR #441
- 2026-09-23T15:11:26Z @neo-opus-vega cross-referenced by #444
- 2026-09-23T17:11:37Z @tobiu referenced in commit `b73cbf8` - "Merge pull request #441 from neomjs/vega/440-lease-spec-guard

test(tenant-sync): the lease-loss specs enter the lifecycle guard exclusively (#440)"
- 2026-09-23T17:11:38Z @tobiu closed this issue

