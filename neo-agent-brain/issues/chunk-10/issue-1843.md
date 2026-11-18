---
id: 1843
title: 'mc-server 2 GB OOM, 1-CPU throttling, and Chroma at-ceiling failure loop'
state: OPEN
labels:
  - bug
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-11-18T15:38:30Z'
updatedAt: '2026-11-18T15:38:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1843'
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
---
# mc-server 2 GB OOM, 1-CPU throttling, and Chroma at-ceiling failure loop

## Context

Live `inspect_deployment`, `get_memory_core_tool_metrics`, `healthcheck`, and container log diagnostics on `2026-11-18` (deployed Brain `d172de48`) during repeated Fleet Manager `fleet timed out · Roster read timed out · plane admission unknown` stalls revealed active resource exhaustion and a failing recovery loop in the containerized plane:

- **`mc-server` V8 Heap OOM (`declaredHeapCeilingMb: 2048`, `cpus: "1.0"`, `65,092` tool calls/24h):** Crashed at `2026-11-18T13:15:11.037Z` (exit `139`) with `Mark-Compact (reduce) 1952.2 (2010.9) -> 1868.3 (1887.7) MB ... FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory`, severing all active agent MCP sessions and FM's shared `planeMailboxClient` session.
- **`mc-server` CPU throttling & slow SQLite/JSON scans:** Measured `98.73%` CPU and `38.24%` cgroup CPU throttling (`16,303 ms` throttled over `122/319` periods in a `33,076 ms` window), driving `who_is_online` (`2,725` calls) to `3,521 ms` avg / `42,547 ms` max, `query_recent_turns` (`1,340` calls) to `5,258 ms` avg / `43,650 ms` max, `list_messages` (`11,384` calls) to `853 ms` avg / `12,747 ms` max, `publish_open_work_copy` to `30,766 ms` max, and `DeploymentStateBridge` direct MCP healthchecks past `30,000 ms` (`10:26:48Z`, `11:14:42Z`). Both `fleet-server` (`cpus: "0.5"`, `35.7%` throttled) and `orchestrator` (`cpus: "1.0"`, `29.1%` throttled) also hit CPU throttling.
- **`chroma` at-cap recovery failure loop (`chromadb/chroma:1.5.9`, `16.99 GB / 17.18 GB` = `88.89%` of its `16 GiB` limit):** `ContainerHealthDiagnosisService` classifies `store-ceiling-exhaustion` (> `80%` threshold) every ~30 s and dispatches `actionClass: 'raise-ceiling'`. `RecoveryActuatorService#raiseComposeServiceCeiling` (`ai/daemons/orchestrator/services/RecoveryActuatorService.mjs:1105-1130`) throws once the memory ceiling is already at its knob maximum, recording `status: 'failed'`, `reasonCode: 'executor-failed'` repeatedly (`5,000` self-heal ledger events: `72` failed, `1,507` deferred).

## The Problem

1. On a 13-seat active fleet (`61,612` memories, `6,073` session summaries, `65k` MC calls/24h), `mc-server`'s `1.0` CPU quota + `2048 MB` heap cap (`deploy/cloud/docker-compose.yml:344,412-413`) combined with heavy/full-table SQLite `json_extract` reads on `who_is_online`, `query_recent_turns`, and `list_messages` pushes `mc-server` into GC thrash, multi-second event-loop stalls exceeding FM's `10 s` `livenessReadTimeout`, and V8 heap OOM crashes.
2. When a `store`-classed service (`chroma`) has already reached its maximum configured ceiling in the knob registry, `raiseComposeServiceCeiling` throws `executor-failed` on every backoff cycle instead of transitioning to alarm-only `recorded` (`ceiling-at-cap`), unlike sibling `pruneEngineBuildCache` (`freedNothing: true` → `recorded` / `prune-freed-nothing` at `RecoveryActuatorService.mjs:1251-1254`).

## The Architectural Reality

- `deploy/cloud/docker-compose.yml`:
  - `chroma` (lines 69–71): `NEO_CHROMA_MEMORY_LIMIT:-8g`, `cpus: "2.0"`.
  - `mc-server` (lines 344–346, 412–413): `NEO_MC_SERVER_HEAP_MB:-768`, `NEO_MC_SERVER_MEMORY_LIMIT:-2g`, `cpus: "1.0"`.
  - `fleet-server` (lines 741–742): `NEO_FLEET_SERVER_MEMORY_LIMIT:-768m`, `cpus: "0.5"`.
- `ai/daemons/orchestrator/services/RecoveryActuatorService.mjs` (`raiseComposeServiceCeiling` lines 1050–1156; `pruneEngineBuildCache` lines 1246–1260) and `ContainerHealthControllerService.mjs`.
- `ai/services/memory-core/` (`who_is_online`, `query_recent_turns`, `list_messages` SQLite queries & in-process memory retention).

Design authority: `RecoveryActuatorService.mjs:1251-1254` ("When the prune frees zero bytes and `statfs` free bytes do not increase, `freedNothing: true` signals the caller to transition the target to alarm-only (`recorded`, `prune-freed-nothing`) rather than burning the window budget on no-op prunes") and #55 / #64 / #73 non-heap-headroom + compose CPU cap contract.

## The Fix

1. **Terminal alarm-only state when `raise-ceiling` hits knob cap:** In `RecoveryActuatorService#raiseComposeServiceCeiling` and `ContainerHealthControllerService`, when the target container is already at or above the knob's maximum allowed ceiling (`violations` indicating cap reached or no higher tier available), return a structured `atCeiling: true` result that transitions the recovery target to alarm-only `recorded` (`ceiling-at-cap`) instead of throwing `executor-failed` indefinitely every ~30 s.
2. **Profile & optimize hot `mc-server` queries + heap retention:** Audit `who_is_online`, `query_recent_turns`, and `list_messages` for full-table `Nodes` JSON scans and high transient JS allocations under concurrent 13-seat traffic, adding targeted SQLite indexes/expressions or bounded read projections to keep p95 latency inside FM's `4 s` axis timeout (#1842 / `rosterAxisTimeoutMs`).
3. **Right-size `mc-server` (and `fleet-server`) CPU/heap headroom:** Increase default/local-agent-os `cpus` and memory/heap ceiling parameters in compose/env configuration (preserving the strict `NEO_MC_SERVER_HEAP_MB` + native/probe headroom < `NEO_MC_SERVER_MEMORY_LIMIT` relationship enforced by `DeclaredHeapCeilings.spec.mjs`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `RecoveryActuatorService#raiseComposeServiceCeiling` (`ai/daemons/orchestrator/services/RecoveryActuatorService.mjs`) | #1251–1254 `pruneEngineBuildCache` alarm-only contract (ADR-0026) | When live limit is already at knob ceiling, return `atCeiling: true` so controller marks `recorded` (`ceiling-at-cap`) without throwing `executor-failed` | Real write/docker lifecycle errors still throw `failed` | JSDoc on `raiseComposeServiceCeiling` | Unit test covering at-cap store target |
| `mc-server` hot SQLite queries & compose resource limits | #55 / #64 / #73 non-heap headroom contract | Indexed/bounded queries for `who_is_online`, `query_recent_turns`, `list_messages` + higher CPU/heap headroom | Existing env overrides (`NEO_MC_SERVER_HEAP_MB`, `NEO_MC_SERVER_MEMORY_LIMIT`) remain respected | JSDoc + compose inline rationale | Query latency benchmark / unit tests + `DeclaredHeapCeilings.spec.mjs` |

Decision Record impact: aligned-with ADR-0014 and ADR-0026.

## Acceptance Criteria

- [ ] Unit test verifies `RecoveryActuatorService` + `ContainerHealthControllerService` transition an already-at-cap `raise-ceiling` target to `recorded` (`ceiling-at-cap`) with zero recurring `executor-failed` retries.
- [ ] Hot SQLite queries (`who_is_online`, `query_recent_turns`, `list_messages`) are benchmarked/indexed so peak `13`-seat load avoids multi-second full-table scans, and compose/heap updates pass `DeclaredHeapCeilings.spec.mjs`.
- [ ] `[L3, post-merge]` After plane rebuild, `mc-server` runs 24h without exit `139` V8 heap OOM and without >10 s `who_is_online` stalls tripping FM's `Roster read timed out` banner under normal fleet load.

## Out of Scope

- Chroma HNSW dead-element bloat remediation (#133, #1432, #1434).
- Host wake receiver 100% CPU spin on stale `primary-session` wakes (#1840).
- Claude async `PostToolUse` sunset-rotation lock (#1841) and `openWorkProducer` serial GitHub queries / mailbox blocking (#1842).

## Avoided Traps

- Raising `NEO_MC_SERVER_HEAP_MB` without simultaneously raising `NEO_MC_SERVER_MEMORY_LIMIT` to keep non-heap + probe headroom below the cgroup ceiling (#55 Avoided Traps: triggers uncatchable kernel `SIGKILL` exit `137`).
- Restarting `chroma` inside `raiseComposeServiceCeiling` (`RecoveryActuatorService.mjs:1144`).

## Related

#55, #64, #73, #133, #1416, #1840, #1841, #1842

---

Live latest-open + closed sweep: checked latest 20 open of 154 (`red` band) + open/closed searches (`mc-server`, `chroma`, `raise-ceiling`, #64, #73, #133, #1416) at 2026-11-18T15:36:25Z; Out of Scope read on #64; no open issue covers the 2 GB `mc-server` OOM + `1.0` CPU throttling + `raise-ceiling` `executor-failed` loop.
A2A in-flight sweep: latest 30 messages read across all read states (`2026-11-18T14:37Z–15:35Z`); no peer claim on `mc-server` resources or `raiseComposeServiceCeiling`.
MC sweep: `"Roster read timed out fleet timed out stale reconnecting plane admission unknown"`, 8 results (#1655/#424/#964 cockpit history only; no prior decision on 2 GB `mc-server` throttling or `raise-ceiling` loop).
Own-assignment sweep: 1 open (`#1230`), non-overlapping.
Structure map (`node ai/scripts/diagnostics/structureMap.mjs --files --loc`, exit 0): `ai/daemons/orchestrator/services`, `ai/services/memory-core`, plus `deploy/cloud/docker-compose.yml`.

unowned-rationale: Ready for a maintainer or contributor; requires live container rebuild/restart for the `[L3, post-merge]` AC.

Origin Session ID: 613064f8-74aa-4baa-8ee7-6a0b8540f31b
Retrieval Hint: "mc-server 2048 heap OOM exit 139 cpu throttling who_is_online chroma raise-ceiling executor-failed"


## Timeline

- 2026-11-18T15:39:31Z @neo-gemini-julia added the `bug` label
- 2026-11-18T15:39:32Z @neo-gemini-julia added the `ai` label
- 2026-11-18T15:39:32Z @neo-gemini-julia added the `performance` label
- 2026-11-18T15:39:32Z @neo-gemini-julia added the `agent-os` label
- 2026-11-18T17:52:01Z @neo-gemini-julia cross-referenced by #1842
- 2026-11-18T17:56:46Z @neo-gemini-julia cross-referenced by PR #1845

