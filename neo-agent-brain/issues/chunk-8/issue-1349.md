---
id: 1349
title: Every closed MCP transport session triggers an all-time summary drift scan
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-10-29T13:46:31Z'
updatedAt: '2026-10-29T13:49:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1349'
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
---
# Every closed MCP transport session triggers an all-time summary drift scan

## Context

A resource sampler reads the live plane's container cgroups every 30 s. Its first segment ran from 2026-10-27 21:33Z to 10-29 12:06Z, 38.5 h. In that window Chroma sent 258 GB to the orchestrator, and 95% of it arrived in bursts with a median gap of 2.5 min. Laying the orchestrator log over the 10-29 samples attributes most of those bursts to one task. Observed on the plane at Brain `436aec63`; the code below is unchanged on `dev` (`2ed12394`).

## The Problem

The `summary` task runs whenever a `SummarizationJobs` marker is pending, and each pass ends with an all-time drift scan that pulls about 170 MB from Chroma. Most markers name transport sessions that own nothing.

- **Markers come from transport closes.** `Server.onSessionClosed` (`ai/mcp/server/memory-core/Server.mjs:709`) calls `SessionService.queueSummarizationJob(sessionId)` for every Streamable-HTTP transport session that closes. The plane holds 28,488 `failed` rows (each at `retry_count` 1) against 4,128 `completed`. The five newest `failed` ids appear in none of the graph's 309,737 nodes. The five newest `completed` ids appear in 2–23 nodes each (memories plus their `SESSION_SUMMARY`).
- **Every pending marker fires a pass at once.** `buildSummaryTrigger` (`ai/daemons/orchestrator/scheduling/summary.mjs:189`) returns a `pending-summarization` trigger whenever `pendingJobs.length > 0`, with no interval gate.
- **Every pass ends with the drift scan.** `ai/scripts/lifecycle/summarize-sessions.mjs` drains the markers (`Pending: N; processed: 0`), then always runs `summarizeSessions()` without an id. That call runs `recoverSessionSummaryReceipts({sessionId: null})` and `findSessionsToSummarize()` across all time, the step the periodic sweep exists for.

**Measured on 2026-10-29:**
- From 13:16:35Z to 13:20:20Z, eight passes ran, each `processed: 0` and `candidates=0`.
- The 30-s receive ticks match pass for pass: 13:20:20 shows 338 MB for two passes, and 13:27:20 and 13:27:50 show 170 MB each for one pass each.
- Each pass also spawns a Node child and initializes `SessionService` and the generation model.

At this rate the loop accounts for roughly 130 GB/day of Chroma reads on our ten-seat plane. Any plane carries the same load in Chroma and orchestrator CPU.

## The Architectural Reality

- **Marker writer:** `SessionService.queueSummarizationJob` (`ai/services/memory-core/SessionService.mjs:1242`). Its JSDoc calls the marker cheap, which holds only for the write.
- **Pending drain:** `SessionService.summarizePendingSessions` claims each id and calls `summarizeSession`, which returns null for a session without memories. The row becomes `failed`.
- **Spawn:** `taskDefinitions.mjs:375` spawns `summarize-sessions.mjs` with static args. The trigger reason reaches `executeFn` (`pipeline.mjs:570`) but not the child, so the child cannot tell a marker pass from a periodic sweep.
- **Marker semantics:** the 4,128 completed rows show that a transport session id sometimes is the Memory Core session id, namely when memories were saved under it. A marker is only worth writing for such a session.

## The Fix

Queue only what can be summarized. `SessionService` records, in process, each session id under which a memory is written. `onSessionClosed` queues a marker only when the closing transport session id is among them, and then forgets it. The drift sweep stays the backstop: if the MC server restarts between the write and the close, the marker is lost, and the next periodic sweep (≤ 10 min) still finds the session.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1: Closing a transport session that wrote no memory leaves `SummarizationJobs` unchanged (unit).
- [ ] AC-2: Closing a transport session that wrote a memory still queues its `pending` marker, and the next pass summarizes it (unit or e2e).
- [ ] Post-merge, once a plane runs the fix: `pending-summarization` passes per day fall by at least 80% against the 10-29 log, and the plane sampler's receive traffic outside the periodic sweep and the tenant sync falls with them.

## Out of Scope

- Letting a marker-triggered pass skip the all-time drift scan. It needs the trigger source to reach the spawned child. If the post-merge reading still shows marker passes costing more than ~5 GB/day, that becomes its own leaf.
- Making `findSessionsToSummarize()` incremental. The periodic sweep (every 10 min) still pays about 170 MB per run; that is a separate, larger change.
- Pruning the 28,488 `failed` rows.
- `github-content-sync`'s full replay at an unchanged head, filed separately.

## Related

neo-agent-brain#81 (another backfill retry loop, different task) · neo-agent-brain#133 · Discussion/rationale: none found

Live latest-open sweep: checked the latest 20 open issues on neomjs/neo-agent-brain at 2026-10-29 13:45Z, none equivalent. A2A sweep: the last 30 messages show no claim on this scope; the defect-note fa159c96 is mine. Memory Core shows no decision on transport-close markers. My own open Brain issues: none on this surface.

Owner: @neo-opus-vega, self-assigned. I hold the measurement, and a fix that lands before the plane's next cut to `dev` gets measured before and after on the same sampler.

Retrieval Hint: "summary pass per closed MCP transport session drift scan Chroma traffic"



## Timeline

- 2026-10-29T13:46:32Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-29T13:46:33Z @neo-opus-vega added the `bug` label
- 2026-10-29T13:46:34Z @neo-opus-vega added the `ai` label
- 2026-10-29T13:46:34Z @neo-opus-vega added the `performance` label
- 2026-10-29T13:55:33Z @neo-opus-vega cross-referenced by PR #1352

