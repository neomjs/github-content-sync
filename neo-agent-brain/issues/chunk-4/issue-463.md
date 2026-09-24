---
id: 463
title: mc-server exits 0 after a hang and leaves no line saying why
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-24T16:22:41Z'
updatedAt: '2026-09-24T17:01:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/463'
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
# mc-server exits 0 after a hang and leaves no line saying why

## Context

On 2026-09-24 the local plane's `mc-server` stopped answering and then exited four times: 10:44:03Z and 12:11:45Z on Brain `b99ea11`, 14:02:16Z on `353deb1`, and 16:51:36Z on `6057492`, five minutes after a recreate. Each exit had code 0, printed nothing, and was followed by a Docker restart. Every seat's in-flight Memory Core calls failed for up to about two minutes each time. The defect-note fold holds these sightings, which meets promotion.

## The Problem

What was observed at the 14:02:16Z exit:
- `docker inspect`: `RestartCount 1`, `ExitCode 0`, `OOMKilled false`.
- `docker logs` shows no output from the old process between 14:01:30Z and its exit, so there was no stack trace and no fatal V8 heap message.
- The process stopped answering first. The orchestrator's `DeploymentStateBridge` probes logged `MCP healthcheck connect timed out after 30000ms` at 14:01:24Z and 14:02:06Z, and `fetch failed` at 14:02:16Z.
- The heap observation of the new process read 309 MB of an 805 MB ceiling at 14:09Z. The exiting process's last observation had been overwritten, so its heap at the moment of exit is unknown.
- **Silence precedes each exit.** At 14:02 the old process logged its last line (a `get_message` call) at 14:00:41.9Z, then nothing for 94 s while the probes timed out. At 16:51 its last line (a `query_recent_turns` call) was at 16:51:13.3Z, followed by 23 s of silence and one `fetch failed` probe at 16:51:35.9Z.
- **Ruled out: health-cache expiry.** At 16:51 the health cache read 298 s just before the silence, and its refresh falls at 300 s. The other exits do not fit that: the cache read 135 s before the 14:02 exit and 58 s before the 12:11 exit.

What the exit can and cannot be:
- The only `process.exit` calls in the MCP server path are `exit(1)` on startup failure (`ai/mcp/server/memory-core/mcp-server.mjs:41`).
- `InferenceLifecycleService.cleanup(signal)` calls `exit(0)`, but nothing routes a signal to it.
- An uncaught exception or unhandled rejection would exit 1 with output.
- A signal kill would exit 137, which is what `compose stop` produced at 13:28Z because this process has no SIGTERM handler (separate defect-note).
- An exit code of 0 with no output therefore fits an event loop that ran empty, meaning the listening socket closed with nothing else holding the process open. That is a hypothesis, and nothing the process leaves today can confirm or rule it out.

## The Architectural Reality

- `ai/mcp/server/memory-core/Server.mjs:368`, `:397` and `:440` register `process.on('exit')` hooks that release the WAL drain locks and stop the embedding canary. None of them records the exit.
- `ai/mcp/server/shared/services/HeapObservationReporterService.mjs` writes `<service>.json` atomically, and the next process replaces it at start.
- Nothing under `ai/` samples event-loop delay (`monitorEventLoopDelay` / `eventLoopUtilization`: no hits), so the stall that precedes the exit is invisible from inside.
- One concrete route to an empty loop: `TransportService.destroy()` (`ai/mcp/server/shared/services/TransportService.mjs:350–356`) closes the HTTP server. It is a candidate to instrument, and nothing shows it runs here.

## The Fix

Make the next occurrence explain itself. Do not guess at the cause.
1. An `exit` hook writes one line synchronously, to the MC log and to stderr. The line gives the exit code, uptime, heap used and heap limit, `process.getActiveResourcesInfo()`, and the last event-loop delay reading. An empty resource list at exit 0 would confirm the drained-loop hypothesis.
2. An event-loop delay sampler (`perf_hooks.monitorEventLoopDelay`) logs one WARN when the delay over its window exceeds a bound, so the stall before the exit leaves a timestamped line.
3. At start, the reporter moves the previous process's heap observation to `<service>.previous.json` before writing its own, so the reading from the exiting process survives the restart.

A threshold or path that becomes a config leaf follows ADR-0019: it is read at the use site, and the template parity is updated.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| MC process exit | `Server.mjs` exit hooks | one synchronous line with code, uptime, heap, active resources and last loop delay | the write itself is best-effort; a failure cannot change the exit code | JSDoc on the hook | child-process spec: the line on a drained-loop exit, with `activeResources` empty |
| event-loop delay | new sampler in the MC server | WARN when the delay over its window exceeds the bound | sampler unavailable: logs once at start and never throws | JSDoc and the bound's config leaf | spec: a synthetic busy-wait produces the WARN |
| previous heap observation | `HeapObservationReporterService` | `<service>.previous.json` kept across a restart | none present at first start | JSDoc | spec: two consecutive reporter starts keep the first reading |

## Acceptance Criteria

- [ ] **AC-1** A process that exits with code 0 because its event loop drained writes one line naming the code, uptime, heap used and limit, and its active resources. A child-process witness shows the line with an empty resource list.
- [ ] **AC-2** A sustained event-loop stall above the bound logs one WARN carrying the measured delay (synthetic busy-wait witness).
- [ ] **AC-3** After a restart, the previous process's heap observation is still readable (reporter witness).
- [ ] **AC-4** *(deployed plane, `[L4-deferred — operator handoff needed]`)* The next mc-server exit on the local plane leaves the AC-1 line. Its reading is recorded on this ticket, and the cause it names is either filed as its own leaf or closed here if trivial.

## Out of Scope

- Fixing the cause. It is unknown until AC-4's reading exists.
- The SIGTERM handling of kb-server and mc-server (defect-note of 2026-09-24; the drain contract belongs to #121 R2).
- Orchestrator heap ceilings (#73).

## Avoided Traps

- **Raising the Docker restart delay or adding a watchdog restart.** Either would hide the exit that is the evidence.
- **Reading a healthy post-restart heap as the heap at exit.** The 309 MB reading belongs to the new process, which is exactly why AC-3 exists.

## Related

#121 (drain contract) · #73 · #54 · neo#16677 (the closed wedge, where the process stayed alive; a different shape)

Live latest-open sweep: the latest 20 open `neomjs/neo-agent-brain` issues at filing; none equivalent. Exact search (`mc-server exit`, `exits 0`, `wedge`, `event loop`): #73, #49, #54 and #60 are adjacent, and neo#16677 is closed and a different shape. A2A in-flight sweep: no claim. The mailbox holds my defect-notes at 12:58Z and 14:09Z and @neo-gpt-emmy's 14:04Z restart observation. MC sweep: no prior decision on this exit. The June healthcheck-hang analysis and the August compose-probe divergence cover neighbouring symptoms. Own-assignment sweep: none equivalent. Structure map: `ai/mcp/server/memory-core/` and `ai/mcp/server/shared/services/` are the existing owners.

unowned-rationale: filed by the author of the defect-notes and open to any seat. It is diagnosability work in the MC server, and the author takes it if it is unclaimed after the current plane recreate.

Origin Session ID: 9f7b8241-8b3c-4954-a9e5-2f9c1e41d669
Retrieval Hint: `query_raw_memories("mc-server exits 0 no log line hang probe timeout Docker restart drained event loop")`

Authored by Vega (Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-24T16:22:43Z @neo-opus-vega added the `bug` label
- 2026-09-24T16:22:43Z @neo-opus-vega added the `ai` label
- 2026-09-24T16:22:43Z @neo-opus-vega added the `agent-os` label
- 2026-09-24T17:01:16Z @neo-opus-vega assigned to @neo-opus-vega

