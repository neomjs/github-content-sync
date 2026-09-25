---
id: 500
title: rem-runs is a declared shared channel that compose keeps per service
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T16:43:52Z'
updatedAt: '2026-09-25T18:02:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/500'
author: neo-opus-vega
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
closedAt: '2026-09-25T17:23:53Z'
---
# rem-runs is a declared shared channel that compose keeps per service

## Context

On the local plane (Brain `94fe224`, recut 2026-09-25) the dream task completed a REM cycle at 16:10:05Z (`orchestrator-state.json`: run `rem-1fdcff41…`, 10 sessions, batch saturated, exit 0) and wrote its receipt into `/app/.neo-ai-data/rem-runs/` inside the orchestrator container, beside 199 older ones. Memory Core's `get_rem_pipeline_state`, read at 16:16Z and 16:31Z: `recentCycles: []`. The lead's 15:33Z read of the same tool concluded "the dream pipeline is dead", and I planned the next plane recreate "behind the first REM cycles" on the same reading. The cycles had landed; the instrument cannot see them.

## The Problem

The Memory Core server reads REM cycle receipts from a directory the orchestrator never writes. Every seat's diagnostic (`get_rem_pipeline_state`) and the Fleet Manager's tasks projection (`ai/services/fleet/fleetTasksSource.mjs:374-403`, which derives its REM rows from `recentCycles`) report no REM activity on a compose plane, whatever REM does. The consolidation watchdog's docblock records the recovery incident's symptom as "`recentCycles: []` alongside a stable ~316-undigested backlog while the orchestrator was alive": the same reading, and at least in part the same cause, because the watchdog runs inside the orchestrator and sees the files while the tool does not.

## The Architectural Reality

- `ai/configBase.mjs:270-273` declares `remRunStateDir` = `<planeDataRoot>/rem-runs` with `planeMember: true`: "Shared directory for REM run-state artifacts. Evolution writes the cycle receipts; Orchestrator liveness and Memory Core health read the same plane coordinate."
- Writer: the dream child the orchestrator spawns (`ai/services/graph/SemanticGraphExtractor.mjs:624/633` for the active-call state; the run-state JSONL through `ai/services/memory-core/helpers/remRunStateStore.mjs`). Readers in the orchestrator process: `ai/daemons/orchestrator/scheduling/remConsolidationLivenessWatchdog.mjs:60-63` and `ai/daemons/orchestrator/services/bootIdentityFactGatherer.mjs:40`. Reader in the mc-server process: `ai/services/memory-core/HealthService.mjs:919-926` (`readRecentRemRunStates({dir: aiConfig.remRunStateDir})`), whose ENOENT branch (`remRunStateStore.mjs:297-299`) returns `[]`. The Golden Path synthesizer runs as an orchestrator child (`ai/daemons/orchestrator/taskDefinitions.mjs:498-501`) and reads the same `HealthService` in that process, so the Golden Path's REM inputs are right; the tool and the wire are not.
- `deploy/cloud/docker-compose.yml` (#425 / PR #428) mounts a private named volume at `/app/.neo-ai-data` on every service (`orchestrator-plane-root` `:497`, `mc-server-plane-root` `:326`) and nests the shared channels on top: `sqlite`, `deployment-state`, `vector-generation`, `handoff`, `heap-observation`, `auth` (`:792-800`). `rem-runs` is not a channel. #425 classified the plane paths from `docker diff`, which shows who writes, and seeded the 202 entries into the orchestrator's private root; the readers were never enumerated.
- `assertPlaneMemberCoherence` (`ai/planeConfig.mjs:345-349`) fails boot when a member does not sit beneath the resolved root. Both containers pass: the coordinate is the same, the medium differs. It is the "alternate realities" #425 named, one member over.
- The `handoff` channel is the precedent: written by the orchestrator's `golden-path` lane, mounted read-only on mc-server (`:328`, `:516-518`).

## The Fix

1. `deploy/cloud/docker-compose.yml`: declare `shared-rem-runs-data`; mount it at `/app/.neo-ai-data/rem-runs` writable on `orchestrator` (beside `shared-handoff-data`) and read-only on `mc-server`, with the WRITE-half / READ-half comments the other channels carry.
2. `test/playwright/unit/ai/deploy/PlaneDataRootMount.spec.mjs`: a fourth arm pinning the channel, mirroring the heap arm at `:76`: the orchestrator mounts it writable, mc-server read-only, no other service mounts it; red when either line is removed.
3. Deploy step on the local plane, inside the recreate I already owe for Brain #492 and #498: seed the new volume from the orchestrator root's `rem-runs` with #425's F1 procedure (graceful stop, `docker cp` tar stream, `up --no-start`, extract, start), so the boot-identity fact and the watchdog keep their history. The shadowed copy under the private root stays until a later cleanup.

**Contract Ledger**

| Target surface | Authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `shared-rem-runs-data` (new), orchestrator writable, mc-server read-only | compose; #425's channel rule | one directory for the writer and both reader processes | a missing mount fails the guard spec, never boot | guard arm red then green; plane receipt on #84 |
| `get_rem_pipeline_state.recentCycles` (existing) | `HealthService.mjs:919` | lists the orchestrator's runs on a compose plane | code unchanged; a plane without the mount still reads `[]` | live read after the recreate |
| `PlaneDataRootMount.spec.mjs` (existing guard, #425 AC-3) | this ticket | fourth arm pins the channel | — | red-first |

**Decision Record impact:** none; aligned with #425's rule (private roots, shared channels nested). No ADR.

## Acceptance Criteria

- [ ] AC-1: compose declares the channel and mounts it writable on the orchestrator and read-only on mc-server; the guard spec's new arm is red with either mount line removed and green with both.
- [ ] AC-2: the guard spec's existing three arms pass unchanged.
- [ ] AC-3 (post-merge, local plane, receipt on #84): after the recreate with the seeded volume, `get_rem_pipeline_state.recentCycles` read from the mc-server process lists the seeded runs, and the first dream cycle after the recreate appears in it.

## Out of Scope

- The window semantics of `undigested` / `digested`: `ai/services/memory-core/managers/ChromaManager.mjs:399-413` reads the first `summarizationBatchLimit` metadatas, and today's reading is 990 + 1,010 = 2,000 of the 2,624 sessions the summaries collection holds. The method docblocks say "undigested-among-recent"; the tool payload does not. A separate leaf if anyone needs the exact backlog.
- Reading the receipts from the graph instead of files: bigger, and `remRunStateStore.mjs` is the designed record.
- `heavyMaintenanceStarvation.leaseStatus: missing` in the healthcheck while a holder is active: the lease file lives in `orchestrator-state`, another private volume. Same class, different reader; noted, not bundled.
- Relocating `rem-runs` under an existing channel (`handoff`, `sqlite`): rejected, it would borrow that channel's semantics and its read/write pairing.

## Related

#425 / PR #428 (the per-service root rule and the seed procedure this extends) · #64 (AC-1 starvation; its AC-6 names the F1 seed of `rem-runs`) · #215 (the REM digestion extraction that made `rem-runs` a plane member) · #84 (the local plane's receipts) · #496 / PR #499 (`fleetGoldenPath` carries the REM counts over the wire) · neomjs/neo-agent-institution#210 (the pane that renders them)

Live latest-open sweep: the latest 20 open issues at 2026-09-25T16:40:29Z; no equivalent (#495 is the picker, #496 the wire read).
A2A in-flight sweep (all read states, last 60 min): claims on #210, #211, #495 and #496; the 15:58Z defect-note names `remRunStateDir` only as a test-fixture gap; none on the channel.
MC sweep: "recentCycles empty mc-server plane root rem-runs" and "REM cycles not landing recentCycles [] dream completed rem-runs orchestrator mc-server volume", 11 results, no prior decision (#425's own sweep recorded none for these paths).
Own-assignment sweep: 7 open; #64 AC-6 names the seed procedure and #425 is closed; none owns the channel.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): owning surface `deploy/cloud/`, sibling precedent the `handoff` channel and `test/playwright/unit/ai/deploy/PlaneDataRootMount.spec.mjs`.

Origin Session ID: d19add67-d33c-489d-99aa-27ad2782ed5e
Retrieval Hint: "rem-runs shared channel compose per-service plane root recentCycles empty mc-server"

## Timeline

- 2026-09-25T16:43:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T16:43:53Z @neo-opus-vega added the `bug` label
- 2026-09-25T16:43:53Z @neo-opus-vega added the `ai` label
- 2026-09-25T16:43:53Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T16:48:10Z @neo-opus-vega cross-referenced by PR #502
- 2026-09-25T17:05:07Z @neo-opus-vega referenced in commit `c0fcfe1` - "ci(brain-unit): the plane-root mount guard joins the executed run list (#500)

The unit job collects every spec and executes only the named list, so the #425 guard and its new REM-channel arm never ran in CI. Eos's RA-1 on PR #502."
- 2026-09-25T17:23:53Z @tobiu referenced in commit `bc6ad07` - "Merge pull request #502 from neomjs/agent/500-rem-runs-channel

fix(deploy): the REM run-state channel is shared between the orchestrator and mc-server (#500)"
- 2026-09-25T17:23:53Z @tobiu closed this issue
- 2026-09-25T17:31:24Z @neo-opus-vega cross-referenced by PR #499
### @neo-opus-vega - 2026-09-25T17:32:09Z

## AC-3 receipt, local plane, 2026-09-25

Recreate at 17:25–17:26Z to Brain dev `bc6ad07` (carries #492, #498 and PR #502). Before `up -d`, `shared-rem-runs-data` was created and seeded from the orchestrator root's `rem-runs` with a `cp -a` copy (200 receipts, count printed by the deploy step).

| Check | Result |
|---|---|
| mc-server mount | `neo-local-agent-os_shared-rem-runs-data -> /app/.neo-ai-data/rem-runs`, `rw=false` |
| receipts visible | orchestrator 200 · mc-server 200 |
| `get_rem_pipeline_state` from mc-server (17:30Z) | `recentCycles`: five completed runs (`rem-1fdcff41` 118.5 s, `rem-d17147f8` 105.0 s, `rem-b53b88b5` 93.5 s, `rem-3b5f0557` 17.5 s, `rem-4dcea090` 111.2 s); `undigested 990 / digested 1010` unchanged, the window by design |
| first dream cycle after the recreate (18:04Z read) | `rem-ae04a6c2`, 138.0 s, `completed`, listed first — the 17:58:26–18:00:44Z run |

AC-3 met on both halves: the seeded runs are listed from the mc-server process, and the first post-recreate cycle appeared in the same read. The cycle itself waited on #504 (the fairness yield abstained instead of dispatching the dream); the plane ran it once the yield threshold was raised by hand, documented in the cut fragment with #504 as the retirement.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-09-25T17:32:20Z @neo-opus-vega cross-referenced by #84
- 2026-09-25T17:54:39Z @neo-opus-vega cross-referenced by #504
- 2026-09-25T18:02:33Z @neo-opus-vega cross-referenced by #64

