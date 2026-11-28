---
id: 1964
title: 'The graph-cache bound thrashes: each message projection stalls 70–85 s'
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - performance
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-27T21:03:12Z'
updatedAt: '2026-11-27T23:33:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1964'
author: neo-opus-ada
commentsCount: 1
parentIssue: 1843
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-27T22:22:24Z'
---
# The graph-cache bound thrashes: each message projection stalls 70–85 s

Owner: Ada (`@neo-opus-ada`), author of the bound (#1960) and owner of the parent #1843.

## Context

The plane was cut to `00762332` (#1960 + #1963) at 20:49:24Z on 2026-11-27. In its first seven minutes, mc-server's event loop blocked for 81.2, 85.3, 70.2 and 83.0 s, with the CPU at 96 % of its one-core quota. The old code (`46228e03`) blocked for 7–21 s in the same post-boot window. `mc_event_loop_block` names the work in flight: the 85.3 s block ending 20:52:28Z held exactly one job, `job:message-graph-projection:2`, and the 70 s and 83 s blocks carry `message-graph-projection` too. Receipts: [#1959 comment](https://github.com/neomjs/neo-agent-brain/issues/1959#issuecomment-7227990698).

The bound was live: `maxGraphNodes` 60000. At 20:52:08Z the cache held 48,024 nodes and 37,090 edges, the collector's 80 % floor.

At 20:58:47Z the operator ran the mitigation. Through the inspector it set the live `GraphService.db.maxGraphNodes` to `null`, while the cache held 92,309 nodes and 485,619 edges. After the block already in flight ended at 20:59:04Z, blocks fell to 1–13 s and the CPU to 8 %.

## The Problem

An eviction takes a node's incident edges with it and clears the vicinity mark at both ends of each one (`Database#collectGarbage`, #1960). The identity hubs have edges to most messages. So each message projection finds its hubs without a vicinity and reloads them whole, which pushes the cache back over the bound, and the next collection evicts them again. Each round costs 70–85 s of synchronous work.

The live fix is undone by any mc-server restart: the next start reads `memoryService.graphCacheMaxNodes` (60000) and the stalls return. An OOM can cause that restart, since the heap leak #1960 closed is open again while the bound is off.

## The Architectural Reality

- Design authority: #1959's acceptance criterion "no OOM or hang". A bound that holds the loop for 70–85 s on every projection fails its "hang" half.
- The bound is the leaf `memoryService.graphCacheMaxNodes` (`ai/configBase.mjs`, env `NEO_MC_GRAPH_CACHE_MAX_NODES`). `GraphService` reads it at boot into `db.maxGraphNodes`. Both of `Database`'s collection paths return early while it is `null` (`ai/graph/Database.mjs`).
- `leaf(null, …, 'number')` is an existing leaf shape (`ai/configBase.mjs`, for example `communityReconciliationMs`), so the change is a default, not a new mechanism (ADR 0019 §5.2).

## The Fix

Default `graphCacheMaxNodes` to `null`, so a restart keeps the cache unbounded. Its docblock says why the bound is off and what re-enables it. The env binding stays, so a plane can still set a bound for a measured experiment. Re-enabling it by default needs a collector that does not strip hub vicinities, or projections that do not need them. That redesign belongs to #1843's open outcome: the leak is back until it lands.

## Acceptance Criteria

- [ ] AC-1: `memoryService.graphCacheMaxNodes` defaults to `null`, and its docblock names the thrash and the condition for turning the bound back on.
- [ ] AC-2: The GraphService boot spec asserts that the leaf defaults to `null` and the cache boots unbounded. (Corrected 2026-11-27: at a `null` default the spec cannot witness that the boot wiring reads the leaf, since null equals null either way. The redesign that turns a default bound back on owes that witness.)
- [ ] AC-3: Post-merge, on a plane cut to the merge commit, no event-loop block of 30 s or more carries `message-graph-projection` in the first hour.

## Out of Scope

- The bounded redesign (#1843).
- The heap ceiling and CPU sizing (#1929).

## Related

Parent #1843 · regression of #1960 (Resolves #1959) · #1929

Live latest-open + closed sweep, 2026-11-27T21:02Z: the latest 20 open; open and closed searches for "graph cache bound", "vicinity eviction", "message projection event loop" and "maxGraphNodes". Closed hits: #1959 (the bound, closed by #1960) and #1958. Open: #1843 (the parent), #1929 (sizing, a different outcome). No equivalent. 153 open (red band), filed as a sub of its owning #1843.
A2A sweep: every message since 20:15Z, all read states. No claim on this scope; Vega's defect-note on the post-cut unresponsive plane is this incident's symptom.
MC sweep: "graph cache bound evicts identity hub vicinity, message projection stalls…", 5 results. Only this bound's own history; no prior decision.
Own-assignment sweep: #1843 is the parent; nothing else covers the bound.

Retrieval Hint: "graph cache bound eviction thrash identity hub vicinity message projection 70-85 s block"

Origin Session ID: 91b0b1e8-37ac-4721-8158-eda48d9e898d

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-27T21:03:12Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-27T21:03:15Z @neo-opus-ada added the `bug` label
- 2026-11-27T21:03:15Z @neo-opus-ada added the `ai` label
- 2026-11-27T21:03:15Z @neo-opus-ada added the `regression` label
- 2026-11-27T21:03:16Z @neo-opus-ada added the `performance` label
- 2026-11-27T21:03:16Z @neo-opus-ada added the `agent-os` label
- 2026-11-27T21:03:20Z @neo-opus-ada added parent issue #1843
- 2026-11-27T21:05:53Z @neo-opus-ada cross-referenced by PR #1965
- 2026-11-27T22:22:24Z @tobiu referenced in commit `637559b` - "fix(memory-core): default the graph-cache bound to off (#1964) (#1965)

With the 60,000-node bound on, each eviction strips the identity hubs'
vicinities, so every message projection reloads them and holds the
event loop 70-85 s (measured after the 00762332 cut). The live process
runs unbounded since the operator's mitigation; this makes a restart
keep it so. The env binding stays for a measured experiment, and the
docblock names what a default bound needs first."
- 2026-11-27T22:22:25Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-27T23:33:16Z

**AC-3 holds.** The plane was cut to `637559b4` (#1965) at 22:31:29Z. In the first hour, 199 event-loop blocks were recorded in `mc_event_loop_block`, from 22:31:32Z to 23:31:25Z, so the window is covered end to end.

| Window 22:31:29Z → 23:31:29Z | Value |
|---|---|
| Largest block | 13.4 s at 23:11:43Z |
| Blocks of 30 s or more | **0** |
| Largest block carrying `message-graph-projection` | 5.1 s |

For contrast, before the live mitigation on the previous cut: 91.8 s at 20:48:01Z, 85.3 s at 20:52:28Z (only `job:message-graph-projection:2`) and 83.0 s at 20:55:16Z.

Source: `get_memory_core_tool_metrics` at limit 1000, filtered to the window by `endedAt`. Its `sinceMs` doesn't bound this list, and a limit of 50 covered only the last 18 minutes. The bound is off by default now (`NEO_MC_GRAPH_CACHE_MAX_NODES` unset in mc-server's env), and the heap sampler restarted on `637559b4`.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


