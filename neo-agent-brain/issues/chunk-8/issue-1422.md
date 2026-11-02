---
id: 1422
title: Fix Brain Unit suite(base) GITHUB_SHA mismatch and wake receiver drain fsync race
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-01T23:31:13Z'
updatedAt: '2026-11-02T00:44:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1422'
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
closedAt: '2026-11-02T00:44:58Z'
---
# Fix Brain Unit suite(base) GITHUB_SHA mismatch and wake receiver drain fsync race

## Context

Diagnosed from `Brain Unit` CI runs [`41003723201`](https://github.com/neomjs/neo-agent-brain/actions/runs/41003723201) (Dependabot PR `#1375`, `2026-10-31T21:44:22Z`) and [`41360208568`](https://github.com/neomjs/neo-agent-brain/actions/runs/41360208568) (PR `#1421`, `2026-11-01T22:25:39Z`), which surfaced two distinct unit-test harness defects in `neomjs/neo-agent-brain`:

1. **`test/playwright/unit/ai/services/knowledge-base/DatabaseService.sync.spec.mjs:22-25` (`checkoutRevision`) permanently breaks `suite (base)` in `.github/workflows/brain-unit.yml`:**
   - In `brain-unit.yml:38-46`, `suite (base)` detaches the checkout to `HEAD^1` (on `pull_request`) or `FETCH_HEAD` (on `push`) while GitHub Actions leaves `process.env.GITHUB_SHA` set to the event's `head` commit.
   - `DatabaseService.sync.spec.mjs:23` computes `process.env.NEO_TEST_BRAIN_REVISION || process.env.GITHUB_SHA || execFileSync('git', ['-C', root, 'rev-parse', 'HEAD'])`, whereas `createBrainImageRevisionReader` (`ai/services/knowledge-base/helpers/imageRevisionReader.mjs:432-440`) calls `resolveCleanBrainCheckoutRevision({root: brain})` (`git -C root rev-parse HEAD`) and rejects when `revision !== resolvedRevision`.
   - Consequently, `DatabaseService.sync.spec.mjs:70` fails all 3 attempts (`Error: Brain tree requires one exact image or checkout revision`) on every `suite (base)` CI run (`unit-report-base` `unexpected: 1`). Worse, because `brain-unit.yml` only fails when a test is green on `base` and red on `head`, permanent failure on `base` masks any future `head` regression in `DatabaseService.sync.spec.mjs:70`.
2. **`test/playwright/unit/ai/daemons/wake/receiver.spec.mjs:1023` (`a dispatch that never settles is named stuck, with its route, after its wake was accepted`) races `ext4` `fsync()` during `'drain'`:**
   - Production `STEP_BOUNDS_MS` (`ai/daemons/wake/receiver.mjs:28-33`) sets `drain: 60_000` (`4×` `accept` and `2×` `dispatch`).
   - `receiver.spec.mjs:1023` sets `bounds = {accept: 250, reload: 250, drain: 250, dispatch: 250}` with `watchdogIntervalMs: 10`.
   - Before `dispatch` runs, `drain` calls `watched.transition(recordKey, 'pending', 'dispatching')` (`ai/daemons/wake/receiverState.mjs:235-282`), which performs two synchronous `fsync()` syscalls (`_writeSynced` + `_syncDirectory`). On Linux `ext4` under parallel CI load, a `> 250ms` journal barrier stall during `transition('pending', 'dispatching')` trips the one-shot watchdog on `'drain'` (`clearInterval(watchdog)` at `receiver.mjs:274`), so `'dispatch'` never trips the watchdog and `receiver.spec.mjs:1101` fails with `Expected: ["dispatch"], Received: ["drain"]`.

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1420` down to `#969`) at `2026-11-01T23:29:51Z`; no equivalent found.
- A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at `2026-11-01T23:29:56Z`; no competing claim on `DatabaseService.sync.spec.mjs` or `receiver.spec.mjs`.
- Memory Core rationale sweep: `query_raw_memories` on `receiver.spec.mjs stuck dispatch drain watchdog DatabaseService.sync.spec.mjs checkoutRevision GITHUB_SHA` returned 10 historical unit-flake records (`#10934`, `#10942`), none covering these two tests.
- Own-assignment sweep: `0` open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-brain`.
- Structure map: `npm run ai:structure-map -- --files --loc` executed; edits stay within existing unit test files under `test/playwright/unit/ai/`.

## The Problem

1. `checkoutRevision(root)` in `DatabaseService.sync.spec.mjs:22-31` gives `process.env.GITHUB_SHA` precedence over `git -C root rev-parse HEAD`, contradicting its own JSDoc (`@summary Uses the checked-out commit as the explicit revision for an unstamped test tree`) and `resolveCleanBrainCheckoutRevision` (`imageRevisionReader.mjs:34-67`).
2. `receiver.spec.mjs:1023` uses a `250ms` ceiling for real `fsync`-backed state transitions (`accept` and `drain`) even in tests that only assert a stuck `dispatch` step, making the one-shot watchdog vulnerable to `> 250ms` Linux `ext4` `fsync()` latency under parallel worker load.

## The Architectural Reality

- `ai/services/knowledge-base/helpers/imageRevisionReader.mjs:34-67` (`resolveCleanBrainCheckoutRevision`) and `:412-470` (`createBrainImageRevisionReader`) require `revision` to match `git -C root rev-parse HEAD` for an unstamped checkout.
- `ai/daemons/wake/receiver.mjs:28-33` (`STEP_BOUNDS_MS`) and `:267-296` (`watchStep`) arm a single `setInterval` watchdog per step (`clearInterval(watchdog)` on first fire).
- Decision Record impact: `none` (test-harness alignment with existing production contracts).

## The Fix

1. In `test/playwright/unit/ai/services/knowledge-base/DatabaseService.sync.spec.mjs`:
   - Resolve `checkoutRevision(root)` via `process.env.NEO_TEST_BRAIN_REVISION`, then `execFileSync('git', ['-C', root, 'rev-parse', 'HEAD'])`, falling back to `process.env.GITHUB_SHA` only if `git rev-parse HEAD` fails.
   - Verify in the spec that `checkoutRevision(aiConfig.neoRootDir)` returns the checked-out `HEAD` even when `process.env.GITHUB_SHA` holds a different 40-hex commit SHA (matching `brain-unit.yml` `suite (base)`).
2. In `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs`:
   - Set default `bounds` for real `fsync`-backed steps (`accept: 2000, reload: 2000, drain: 2000, dispatch: 250`) while allowing `start({stepBoundsMs})` to override `accept: 250` in the mocked `accept: never` test.
   - Add a deterministic regression check verifying that a `300ms` state transition inside `drain` does not preempt a stuck `dispatch` step (`Expected: ['dispatch']`).

## Acceptance Criteria

- [ ] AC-1: `DatabaseService.sync.spec.mjs` passes when `process.env.GITHUB_SHA` is set to a 40-hex SHA that differs from the checked-out `HEAD` (the `brain-unit.yml` `suite (base)` detached-`HEAD^1` state).
- [ ] AC-2: `receiver.spec.mjs` `a dispatch that never settles is named stuck, with its route, after its wake was accepted` passes even when `state.transition` takes `300ms` during `'drain'`, and all 47 `receiver.spec.mjs` arms remain green.

## Out of Scope

- Modifying production `STEP_BOUNDS_MS` in `ai/daemons/wake/receiver.mjs` or `resolveCleanBrainCheckoutRevision` in `ai/services/knowledge-base/helpers/imageRevisionReader.mjs`.

## Avoided Traps

- Increasing Playwright `retries` or skipping the specs in CI: masks real regressions instead of fixing the root-cause test bugs.

## Related

- PR `#1375` / `#1421` (where run `41003723201` and `41360208568` surfaced both failures).

Origin Session ID: d182e09c-94c2-4095-abe3-330db2e7ce00
Retrieval Hint: "DatabaseService.sync.spec.mjs checkoutRevision GITHUB_SHA suite base HEAD^1 receiver.spec.mjs drain fsync dispatch stuck"


## Timeline

- 2026-11-01T23:31:14Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-01T23:31:14Z @neo-gemini-julia added the `bug` label
- 2026-11-01T23:31:15Z @neo-gemini-julia added the `ai` label
- 2026-11-01T23:31:15Z @neo-gemini-julia added the `testing` label
- 2026-11-01T23:31:15Z @neo-gemini-julia added the `agent-os` label
- 2026-11-01T23:35:47Z @neo-gemini-julia cross-referenced by PR #1423
- 2026-11-02T00:02:48Z @neo-gemini-julia referenced in commit `83a6a33` - "test(brain-unit): move GITHUB_SHA override into try and wait past max step bound (#1422)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>"
- 2026-11-02T00:44:58Z @tobiu referenced in commit `3622197` - "test(brain-unit): fix suite(base) GITHUB_SHA mismatch and wake drain fsync bound (#1422) (#1423)

* test(brain-unit): fix suite(base) GITHUB_SHA mismatch and wake drain fsync bound (#1422)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>

* test(brain-unit): move GITHUB_SHA override into try and wait past max step bound (#1422)

Co-Authored-By: Julia <neo-gemini-3-1-pro@neomjs.com>"
- 2026-11-02T00:44:58Z @tobiu closed this issue

