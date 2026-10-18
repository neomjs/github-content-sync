---
id: 1158
title: FleetLifecycleService.spec leaks its stub seams into later spec files
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T22:04:52Z'
updatedAt: '2026-10-18T22:52:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1158'
author: neo-opus-grace
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
closedAt: '2026-10-18T22:52:58Z'
---
# FleetLifecycleService.spec leaks its stub seams into later spec files

## Context

Promoted from `@neo-gemini-pro`'s defect-note (21:55Z): `onboardPeer.spec.mjs:553` flaked in CI on `#1154` (`0c8aaeec`), a PR that touches neither spec. Reproduced on `origin/dev` @ `9cd1528b`, as observation rather than inference:

- `npm run test-unit -- test/playwright/unit/ai/FleetLifecycleService.spec.mjs test/playwright/unit/ai/scripts/fleet/onboardPeer.spec.mjs --workers=1` → 1 failed, 204 passed. The failure is `TypeError: lifecycle.getLaunchAdmission(...)?.revocationMark is not a function` at `FleetManager.startAgent` (`ai/services/fleet/FleetManager.mjs:431`).
- `onboardPeer.spec.mjs` alone → 37 passed.

It flakes in CI only when the two files land in the same worker.

## The Problem

`FleetLifecycleService.spec.mjs` resets the singleton **before** each test (`install()`), and nothing restores it **after** the file's last test. Each stub that test leaves on the singleton (spawn, process table, signals, windows, registry, the launch-admission issuer) is what the next spec file in that worker reads. The launch-admission describe's issuer stub has no `revocationMark`, and `FleetManager.startAgent` calls it, so `onboardPeer` fails. The other stubs leak the same way and can fail any later Fleet spec that runs after it in the worker.

The leaking helper is mostly mine (`installDesktopSeat`, `2d839fc1`, 2026-10-07). `FleetManager.mjs:431` started reading `revocationMark` on 2026-10-08 (`6de77a36`), which turned a latent leak into a failure.

## The Architectural Reality

- `install()` (`test/playwright/unit/ai/FleetLifecycleService.spec.mjs:182-226`) writes 27 `FleetLifecycleService` seams plus `McpLaunchAdmissionService.handoffPromise` / `continuityPhase`. The file runs `test.describe.configure({mode: 'serial'})`, and it has no file-level `afterAll`.
- `installDesktopSeat()` (`:3608`) replaces `FleetLifecycleService.launchAdmission` with `{continuityPhase, activate, revoke, holds, statusOf}`. The describe's last arm (`:4448`) leaves that stub in place.
- `FleetManager.startAgent` reads `lifecycle.getLaunchAdmission?.()?.revocationMark(agentId)` (`FleetManager.mjs:431`). `null` means the real issuer, and the real issuer has the method.

**Design authority:** the spec's own reset comment (`:201-202`): *"so a collision test cannot bleed into the next serial sibling (singleton-stateful service)"*. The file intends no bleed and covers only its own siblings.

## The Fix

In `FleetLifecycleService.spec.mjs` only:
1. Make `install()` the single list of seams it owns: one object of seam values assigned in one place, so the restore set derives from it and cannot drift from it.
2. Capture the originals of exactly those seams, and of the two `McpLaunchAdmissionService` fields, before the first `install()`.
3. Add a file-level `test.afterAll` that clears the process records (with their timers) and assigns the originals back.

No production change: a stub missing a method is a test-isolation defect, not a reason for `?.` at the call site (ADR-0019 B3's reasoning applies).

## Acceptance Criteria

- [ ] AC-1: The two-file command above passes in one worker: 0 failed, `onboardPeer.spec.mjs:553` green. It fails on `origin/dev` (red-first, recorded in the PR).
- [ ] AC-2: After `FleetLifecycleService.spec.mjs` finishes, every seam `install()` writes holds the value it held before the file ran, `launchAdmission` among them, and the restore set is derived from `install()`'s own list rather than kept as a second one.
- [ ] AC-3: `FleetLifecycleService.spec.mjs` and `onboardPeer.spec.mjs` pass on their own, and the Brain unit CI is green.

## Out of Scope

- `McpLaunchAdmissionService`'s maps (`generations`, `revocations`). Arms mutate them through the real service; no cross-file failure is reproduced from them.
- Other spec files' teardown. A suite-wide singleton reset is a separate design question.
- Making the stub issuer complete. Without the restore, a complete stub would still leak its behavior.

## Related

- Refs `#1154` (where it surfaced). The same describe gains arms in PR `#1156`, which inherits this restore once both land.

Live latest-open sweep: checked the latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T22:04:23Z` (newest `#1157`), plus a search for `FleetLifecycleService.spec leak OR onboardPeer flake OR revocationMark`; no equivalent.
A2A claim sweep: `list_messages({status: 'all'})` at `2026-10-18T22:04Z`; the defect-note `MESSAGE:61bdcdec` is a capture, and nobody has claimed it.
MC sweep: `query_raw_memories` on the symptom ("onboardPeer startAgent failed", "spec leaks singleton stub into the next file"). It found `@neo-gemini-pro`'s matching trace and `@neo-opus-vega`'s 2026-09-27 sighting of the same arm red on dev (then untriaged). The `#518` it names is a different, closed defect. No prior decision.
Own-assignment sweep: 4 open (`#1157`, `#1145`, `#684`, `#54`), none overlapping.

Origin Session ID: 874e962d-26d5-415d-99e7-582d2b05e15c

Retrieval Hint: `query_raw_memories("FleetLifecycleService.spec install leaks stub launchAdmission revocationMark onboardPeer cross-file worker")`


## Timeline

- 2026-10-18T22:04:53Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T22:04:54Z @neo-opus-grace added the `bug` label
- 2026-10-18T22:04:54Z @neo-opus-grace added the `ai` label
- 2026-10-18T22:04:54Z @neo-opus-grace added the `testing` label
- 2026-10-18T22:04:54Z @neo-opus-grace added the `agent-os` label
- 2026-10-18T22:10:33Z @neo-opus-grace cross-referenced by PR #1159
- 2026-10-18T22:50:54Z @neo-opus-grace cross-referenced by PR #1156
- 2026-10-18T22:52:58Z @tobiu referenced in commit `a3bc805` - "fix(test): FleetLifecycleService.spec hands its stub seams back after its last test (#1158) (#1159)

install() reset the singleton before each test and nothing restored it after
the file's last one, so its stubs were what the next spec file in the worker
read. The launch-admission describe's issuer stub has no revocationMark, and
FleetManager.startAgent calls it: onboardPeer.spec:553 failed whenever CI put
the two files in one worker.

install()'s seams are now one object (testSeams), their originals are read off
its keys before the first test, and a file-level afterAll assigns them back
with the two McpLaunchAdmissionService fields. Spec-only; no production change.

Red on origin/dev 9cd1528b (pair in one worker: 1 failed, 204 passed); green
here (205 passed). FleetLifecycleService.spec alone: 170 passed."
- 2026-10-18T22:52:59Z @tobiu closed this issue
- 2026-10-18T23:07:12Z @neo-opus-ada cross-referenced by #1164

