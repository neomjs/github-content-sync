---
id: 946
title: 'Runtime retirement kept the reference lease from 02:51:27Z and never freed it'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T03:29:03Z'
updatedAt: '2026-11-13T05:06:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/946'
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
closedAt: '2026-11-13T05:06:26Z'
---
# Runtime retirement kept the reference lease from 02:51:27Z and never freed it

## Context

On 2026-11-13 the installed FM (e717bbf, Brain 857fe415) began a runtime-retirement pass at 02:51:27Z that never finished. It holds two leases, both owned by pid 99017 (FM main) and pulsed once at 02:51:28.005Z:

- `~/Library/Application Support/neo-harness/installer/package.lock` (`neo-harness-installer`, startedAt 02:51:27.998Z)
- `~/Library/Application Support/neo-harness/brain/runtime/.runtime-references.lock` (`fleet-runtime-references`, startedAt 02:51:27.999Z)

Every seat Start waits on the second (neomjs/neo-agent-brain#1667), so no stopped seat can start. Every FM install waits on the first until the FM process ends.

Observed 03:20–03:31Z: FM main alive at 0.0–0.1% CPU. Its only open file under `runtime/` is its host pin, and its only children are the Fleet and three Electron helpers, so no `ps`/`lsof` census child is running. There is no `.retiring-*` entry. main.log, which tees `log`/`warn`/`error` (`harness/mainLog.mjs`), has no `HARNESS_RUNTIME_*` line after 01:14:55Z. `runtime/` holds five generations; live processes run from four (7c431ce 153, 38c19f0 44, e1f968b 18, 078cb1b 2), and 1bbebbbb has none, so it is the likely candidate the pass was on.

Ruled out at 03:40–03:52Z:
- A named pipe in a payload: none exists under `runtime/`, and `inventoryRuntimePayload` refuses non-file entries without opening them.
- A blocked libuv pool: `sample` shows all four FM-main `libuv-worker` threads idle in `__psynch_cvwait`.
- `assertRuntimeBindings` / `readSealedRecord`: plain fs awaits.
- A payload that cannot be verified: a read-only `readRuntimeGeneration` of 1bbebbbb completed in 5984 ms.

`assertCurrent()` pulses both leases at every step boundary. A `lastPulse` frozen 6 ms after acquisition therefore places the stall at the first await after the cheap per-generation checks: the payload verify or the process census. Static reading does not name which.

## The Problem

`harness/runtimeMaintenance.mjs#retireRuntimeGenerations` takes both leases and releases both in one `finally`. Both are still held, so **the pass never settled**: it is suspended inside its `try` (inside an await), before any rename, with no I/O or child in flight. Vega found the second lease; it rules out the other reading, a lease that outlived its pass.

Consequences while it hangs:
- `startRuntimeMaintenance` skips every later tick, because one pass is pending.
- A graceful FM quit hangs (code path, not tried live): `will-quit` calls `preventDefault()`, then `teardownBrain` awaits `runtimeMaintenance.close()`, which awaits the pending pass, before it stops the Fleet. So no continuity handoff runs and `appLifecycle` has no watchdog. `verifySeats` also refuses a handoff while any Start is pending.
- Seat Starts and FM installs block until the FM process ends.

## The Architectural Reality

- `retireRuntimeGenerations`: `transactionFn()` (package lease) → `references.holdRuntimeGenerationReferences` → per generation `preliminarilyRetained` → `verifier.readRuntimeGeneration` (inventories the whole payload) → `inspectProcesses` (10 s exec timeouts) → rename / `finishRemoval` / `removeTree` → `finally` releases both.
- `startRuntimeMaintenance(run, {intervalMs: 60000})`: non-overlapping; `close()` awaits the pending pass.
- `assertCurrent()` pulses both leases and throws when a lease is missing or carries another token, and `release()` unlinks only its own token. So removing the references lease cannot let a resumed pass rename or delete a generation, or remove a successor's lease.
- `removeOwnedRuntimeTree` deletes by path. Its inode check happens at entry, so it cannot protect a tree that a successor retains or replaces while an abandoned removal is still running.

## The Fix

1. Bound the pass's reads: race each payload verify and census against a per-pass deadline and the owner's abort, so the `finally` runs. Both leases release, and the result names the step and generation. The FM logs `HARNESS_RUNTIME_MAINTENANCE_DEFERRED <reason> <step> <generation>` once per halted pass.
2. Never abandon a removal. A removal that has started finishes under both leases, and the halt takes effect at the next step.
3. Keep the pulse progress-only. `assertCurrent()` already pulses at each step boundary, so a stuck step leaves `lastPulse` frozen, which neomjs/neo-agent-brain#1667 can read as a dead holder. A timer pulse would make a hung pass look alive.
4. Teardown aborts the running pass and joins it for at most `closeTimeoutMs`, so an FM quit cannot hang behind a stuck pass.

## Acceptance Criteria

- [ ] A pass whose read never settles releases both leases at its deadline, and its result names the step and generation: hangs on base, releases at head.
- [ ] During a stuck step, the lease's `lastPulse` does not advance.
- [ ] Teardown completes while a pass is stuck: the pass is aborted and `close()` returns within its bound.
- [ ] A halt during a removal keeps both leases until the removal finishes, so no successor can retain or replace the tree under it.

## Post-Merge Validation

- [ ] FM build + install, then a seat Start succeeds while retirement runs.
- [ ] The await that hung on 2026-11-13 is named by the first `HARNESS_RUNTIME_MAINTENANCE_DEFERRED pass-deadline` line after install, or this ticket records that none recurred.

## Unblock on the live FM (operator; done outside this ticket)

1. Done: a Stop on Grace's seat cancelled the pending Starts (the operator's at 02:56Z and my retry at 03:18Z). The waiters' once-a-second writes to `runtime/` stopped at 03:30:53Z, and her record read `off` with no pending action.
2. Done 03:38:21Z: the operator removed only `runtime/.runtime-references.lock`; `installer/package.lock` stays.
3. Done: Grace started at 03:38:38Z (pid 73900, child of the Fleet). Her record reads `ok`, launch admission is `active`, all four MCP servers are `active`, and her Start released the references lease again.
4. Still open: this FM (pid 99017) cannot quit gracefully, so the install that ships this fix must end it by force. Expect an unclean issuer exit: seats keep running and go stale, and each rebinds with a Stop + Start. The dead pid makes `package.lock` reclaimable for the installer. Plan that install with the operator; until then, no FM restart.

## Out of Scope

- The Start-side bounded wait and named refusal: neomjs/neo-agent-brain#1667.
- A removal that itself never settles: it keeps custody by design. The observed hang came before any rename.

## Related

neomjs/neo-agent-brain#1667 · defect-note `9fbaad41905f3d08` · #7

Live latest-open + closed sweep: latest 20 open in Institution and Brain, plus an all-state search (runtime-references, retireRuntimeGenerations, runtime retirement, reference lease) at 2026-11-13T03:35Z; no equivalent. A2A: none on this scope. Backlog band: Institution 60 open (red); no owning epic for runtime maintenance.

Origin Session ID: 2893c284-25f8-411c-b5dc-f37eafd573fe
Retrieval Hint: "runtime retirement pass held runtime-references lease, FM main idle"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T03:29:03Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T03:29:05Z @neo-opus-ada added the `bug` label
- 2026-11-13T03:29:05Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T03:29:06Z @neo-opus-ada added the `ai` label
- 2026-11-13T03:33:01Z @neo-opus-ada cross-referenced by #1667
- 2026-11-13T03:55:50Z @neo-opus-ada cross-referenced by PR #947
- 2026-11-13T04:29:38Z @neo-opus-ada referenced in commit `f1dc3f4` - "fix(harness): a halt never abandons a runtime removal in flight (#946)

The deadline and the owner's abort raced every await, the path-based
removeTree included. A halt mid-removal released both leases while the
deletion kept running, so a successor that retained or replaced the
quarantined tree could still lose it to the old operation.

Only reads (payload verify, census) race the halt now. A removal that
has started finishes under both leases, and the halt takes effect at
the next step."
- 2026-11-13T05:06:26Z @tobiu referenced in commit `ca56061` - "feat(harness): a stuck retirement pass frees its leases (#946) (#947)

* feat(harness): a stuck retirement pass frees its leases (#946)

A runtime-retirement pass held installer/package.lock and
runtime/.runtime-references.lock across unbounded awaits. On 2026-11-13
one never settled: every seat Start and every FM install waited until
the FM process ended, and a graceful quit hung in close().

Each await in retireRuntimeGenerations now races the pass's deadline
(2 min, about ten real passes) and the owner's abort. The finally
releases both leases and the result names the step and generation.
Teardown aborts the running pass and joins it within closeTimeoutMs.
main.mjs passes the signal and logs a halted pass once.

* fix(harness): a halt never abandons a runtime removal in flight (#946)

The deadline and the owner's abort raced every await, the path-based
removeTree included. A halt mid-removal released both leases while the
deletion kept running, so a successor that retained or replaced the
quarantined tree could still lose it to the old operation.

Only reads (payload verify, census) race the halt now. A removal that
has started finishes under both leases, and the halt takes effect at
the next step."
- 2026-11-13T05:06:27Z @tobiu closed this issue

