---
id: 1142
title: 'Fleet boot slowed 10–20× today, past the FM shell''s 15 s window'
state: OPEN
labels:
  - bug
  - ai
  - regression
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-18T18:55:00Z'
updatedAt: '2026-10-18T20:10:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1142'
author: neo-opus-vega
commentsCount: 2
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
---
# Fleet boot slowed 10–20× today, past the FM shell's 15 s window

## Context

Operator, 2026-10-18: a boot that "now takes TWICE as long" is product damage. That evening the installed Fleet Manager opened three times with an empty roster.

**Observed, read-only, across 77 plane-attach boots in the FM's `main.log`.** The span measured runs from the shell's `HARNESS_BRAIN_PLAN` line to the Fleet's `seams bound … viewer verified plane-side` line:

| Boots | Span |
|---|---|
| All 70 boots, 2026-09-26 → 10-16 | 0.3–2.3 s (most 0.5–1.0 s) |
| First four on today's install (Institution `2f9056a`, Brain `d6a905dc`), 15:40–15:57Z | 9.0 / 9.7 / 7.8 / 8.7 s |
| 18:07Z, 18:36Z | 13.8 s, 16.8 s |

Both evening boots logged `HARNESS_BRAIN_BOOT_FAILED fleet transport not ready within 15000ms`. The Fleet came up 11 s and 1.8 s later, but the UI never attached, so there was no roster. The resulting update trap is tracked in the epic this exploration feeds.

**Measured component.** `readRuntimeGeneration` re-inventories and SHA-256-hashes the whole runtime payload. It runs on every packaged Fleet boot via `bindCurrentRuntime`. On the installed runtime (23,727 files, 965 MB) on an idle machine it took **4.9 s cold and 2.9 s warm**. This step is new since #1051; the 10-14 build (Brain `e219267`) had no `bindCurrentRuntime`.

**Unmeasured remainder.** That leaves ≈4–5 s at 15:57Z and ≈9–12 s at 18:36Z for `resolveFleetViewerClaim` plus `planeClient.init`. During the slow boots, the Memory Core server process sat at 85–102 % CPU with **0 established connections**. Its HealthService served a cache aging 40 s → 267 s without refreshing, and logged nothing except health probes (18:43–18:47Z).

**Inference, not yet measured:** two causes are stacked. One is a new constant cost, the full payload re-hash on every boot. The other is load-dependent: Memory Core's event loop saturated by internal work, delaying every plane request.

## The Problem

The shell gives the Fleet 15 s, after which the roster stays empty for the session. The same payload verification also runs inside the continuity handoff's single 10 s deadline: once per runtime key in `handoffRuntimeCustody`, under `performHandoff`'s one race (see #1081). At 18:05Z survivor validation used 9.2 s of that 10 s before it refused. It runs again in boot recovery. So this regression does more than slow a boot: it pushes restarts and updates over hard limits.

## The Architectural Reality

- `ai/services/fleet/devFleetServer.mjs` (≈:100–168): `bindCurrentRuntime` → `resolveFleetViewerClaim` → `planeClient.init` → the `seams bound` line.
- `ai/services/fleet/runtimeGeneration.mjs`:
  - `readRuntimeGeneration` (:180) compares a fresh `inventoryRuntimePayload` (:87, walks and hashes every file) against the record's `payloadDigest`.
  - `bindCurrentRuntime` (:266) calls it on every boot.
- Other callers: `recoverRuntimeCustodyUnderGuard` (`verifyPin`) and `handoffRuntimeCustody` (`verifySeats`), both in `FleetLifecycleService.mjs`.
- The installer already verifies the payload when it prepares the generation (Institution `harness/install.mjs`, "prepare verified runtime").
- Memory Core runs as a single Node process in `neo-local-agent-os-mc-server-1`. Any CPU-bound work on its event loop delays every request.

Design authority: the operator's line above, and the FM shell's 15 s readiness window (Institution `harness/brain.mjs`, `awaitFleetReady`). That window is the budget the product already enforces, and 70 consecutive boots met it at ≤ 2.3 s.

## The Fix (exploration)

1. **Phase timings in the boot log.** Runtime bind, viewer claim, plane init, recovery and transport each get one line with milliseconds, so the split is read, not inferred.
2. **A CPU profile of the Memory Core server while it is saturated** (before any restart), naming the consumer.
3. **A measured breakdown and the fix leaves.** Each component of today's 8–17 s is set against the ≤ 2.3 s baseline, and each fix is filed as a leaf with a budget. Candidates: bind the boot to the install-time verification plus a cheap identity check (record, inode, size, mtime) instead of a full re-hash; take Memory Core's background work off the request path.

## Acceptance Criteria

- [ ] AC-1: Fleet boot logs per-phase milliseconds for runtime bind, viewer claim, plane init, recovery and transport (unit or source receipt).
- [ ] AC-2: The Memory Core saturation consumer is named by a CPU profile, or equivalent evidence, captured while saturated and posted here.
- [ ] AC-3: A breakdown of the 8–17 s against the ≤ 2.3 s baseline is posted, with one measurement per component, and the fix leaves are filed with budgets.

## Out of Scope

- The fixes themselves: they are the leaves AC-3 files.
- The FM shell giving up at 15 s with no later attach, and the update trap: tracked under the "An FM update never requires closing a harness" epic.

## Related

#1051 (introduced `bindCurrentRuntime`), #980, #1081 (handoff deadline), #1138 / #1139, #1140 / #1141.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-18T18:52Z; nearest #1081 (shutdown budget), not equivalent.
Org search ("boot slow", "BOOT_FAILED", "fleet transport not ready", "readRuntimeGeneration", "payload digest boot", "Memory Core CPU"): only #980 (closed), no equivalent.
A2A and Memory Core sweeps: not runnable from this seat (started outside the FM, no MCP servers); the org search above substitutes.
Structure map: `ai/services/fleet`, `ai/mcp/server/memory-core` (existing files).

Origin Session ID: c092b691-0e97-4623-b91a-86a8c9ca2e84
Retrieval Hint: "fleet boot regression bindCurrentRuntime readRuntimeGeneration payload re-hash plane viewer verification BOOT_FAILED 15000ms"

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-18T18:55:02Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T18:55:02Z @neo-opus-vega added the `bug` label
- 2026-10-18T18:55:02Z @neo-opus-vega added the `ai` label
- 2026-10-18T18:55:03Z @neo-opus-vega added the `regression` label
- 2026-10-18T18:55:03Z @neo-opus-vega added the `performance` label
- 2026-10-18T18:55:03Z @neo-opus-vega added the `agent-os` label
- 2026-10-18T18:55:27Z @neo-opus-vega cross-referenced by #732
- 2026-10-18T18:56:56Z @neo-opus-vega cross-referenced by #1143
- 2026-10-18T18:56:59Z @neo-opus-vega cross-referenced by #733
- 2026-10-18T18:57:00Z @neo-opus-vega cross-referenced by #734
### @neo-opus-vega - 2026-10-18T19:08:14Z

Evidence from the 19:03–19:08Z recovery (read-only except the operator-approved Memory Core restart):

- **Memory Core ignored SIGTERM for the full stop grace.** `docker restart -t 60` took 61 s; it was killed, not stopped. Fits an event loop blocked by CPU-bound work.
- **The startup health check alone took 35 s** (`Performing fresh health check` 19:04:54Z → `[Startup] Memory Core health check passed` 19:05:29Z, 48,640 memories / 4,652 summaries).
- **CPU stays high after the restart: 74–78 % at +45 s, 98 % at +60 s.** So the burn is startup or background work that restarts with the process, not a stuck loop that a restart clears.
- **First boot of the newly installed FM** (Brain `9cd1528b`, fresh runtime generation `d4d533a9…`): `HARNESS_BRAIN_PLAN` 19:07:16.634Z → viewer verified 19:07:25.794Z (9.2 s) → transport 19:07:25.881Z. Inside the shell's 15 s window this time, still about 10× the 70-boot baseline.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-18T19:16:39Z @neo-opus-vega cross-referenced by #1144
### @neo-opus-vega - 2026-10-18T20:10:24Z

**Handover (Vega, session sunset 2026-10-18).** The exploration is open and no code is written. Evidence so far: the body, plus [6253940045](https://github.com/neomjs/neo-agent-brain/issues/1142#issuecomment-6253940045). That comment records the Memory Core restart: it ignored SIGTERM for 60 s, its startup health check took 35 s, and CPU was back at 98 % within a minute.

Pickup, in order:
1. **AC-1.** Add per-phase ms lines to the `devFleetServer` boot (runtime bind, viewer claim, plane init, recovery, transport).
2. **AC-2.** Profile Memory Core while it is saturated. It re-saturates within a minute of a restart, so any moment works (Node inspector inside `neo-local-agent-os-mc-server-1`).
3. **`bindCurrentRuntime` fix candidate.** Trust the installer's prepare-time verification plus a cheap identity check (record, inode, size, mtime) instead of a full 965 MB re-hash per boot. The same call also sits inside the 10 s handoff deadline (#1081).

Owner: Vega. Another seat may take it; say so in A2A first.

— Vega (Claude Opus 5.5, Claude Code) 🌿



