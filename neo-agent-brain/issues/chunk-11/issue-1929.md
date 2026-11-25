---
id: 1929
title: Right-size mc-server CPU and heap headroom from post-#1843 query profile
state: OPEN
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees: []
createdAt: '2026-11-25T17:03:51Z'
updatedAt: '2026-11-25T17:03:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1929'
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
blockedBy:
  - '[ ] 1843 mc-server 2 GB OOM, 1-CPU throttling, and Chroma at-ceiling failure loop'
blocking: []
---
# Right-size mc-server CPU and heap headroom from post-#1843 query profile

## Context

Split from #1843 (Fix 3) per `@neo-opus-ada`'s intake (`https://github.com/neomjs/neo-agent-brain/issues/1843#issuecomment-7168633243`).

On `2026-11-18` (`13:15Z`) and `2026-11-25` (`14:24Z`), `mc-server` crashed with V8 heap OOM (`Ineffective mark-compacts near heap limit` around ~`1.9–2.0 GB` inside a `3 GiB` local override / `2g` compose default) and suffered `38.24%` cgroup CPU throttling under `cpus: "1.0"` (`NanoCpus: 1e9`). Both `fleet-server` (`cpus: "0.5"`, `35.7%` throttled) and `orchestrator` (`cpus: "1.0"`, `29.1%` throttled) also showed sustained CPU throttling under 13-seat swarm load.

## The Problem

Sizing `mc-server`'s CPU quota, `NEO_MC_SERVER_HEAP_MB`, and `NEO_MC_SERVER_MEMORY_LIMIT` *before* #1843 eliminates the synchronous `860 ms` `who_is_online` floor and `246 s` `list_messages` `box: 'all'` scans would bake query-bug waste into permanent container caps. Once #1843 lands, `mc-server`'s steady-state CPU and V8 heap working set under 13-seat load must be re-measured and `deploy/cloud/docker-compose.yml` defaults (`NEO_MC_SERVER_HEAP_MB:-768`, `NEO_MC_SERVER_MEMORY_LIMIT:-2g`, `cpus: "1.0"`) right-sized while preserving the non-heap RSS headroom invariant from #55 / #73.

## The Architectural Reality

- `deploy/cloud/docker-compose.yml`:
  - `mc-server` (lines 344–346, 412–413): `NEO_MC_SERVER_HEAP_MB:-768`, `NEO_MC_SERVER_MEMORY_LIMIT:-2g`, `cpus: "1.0"`.
  - `fleet-server` (lines 741–742): `NEO_FLEET_SERVER_MEMORY_LIMIT:-768m`, `cpus: "0.5"`.
  - `orchestrator`: `cpus: "1.0"`.
- `test/playwright/unit/ai/daemons/orchestrator/DeclaredHeapCeilings.spec.mjs`: enforces the non-heap headroom relationship between `NEO_MC_SERVER_HEAP_MB` and `NEO_MC_SERVER_MEMORY_LIMIT`.
- Mandatory gate: read `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` (`ADR-0019`) before touching any `ai/` config or env-var surface.

Design authority: #55 / #73 `headroom-for-non-heap-rss` invariant (`ceiling + observedNonHeapBytes < cgroup limit`, minimum `256 MiB+` non-heap margin so V8 heap pressure throws a catchable heap-limit error rather than triggering an uncatchable kernel `SIGKILL` exit `137`).

## The Fix

1. After #1843 merges and deploys, measure `mc-server` post-fix CPU throttling ratio, V8 old-space peak, and non-heap RSS under active 13-seat load (`inspect_deployment` + `get_memory_core_tool_metrics`).
2. Right-size `mc-server` (and, if still throttled > `25%`, `fleet-server` / `orchestrator`) CPU and `NEO_MC_SERVER_HEAP_MB` + `NEO_MC_SERVER_MEMORY_LIMIT` defaults in `deploy/cloud/docker-compose.yml` together, keeping `DeclaredHeapCeilings.spec.mjs` green.
3. Verify the 24 h `[L3, post-merge]` stability run on the rebuilt plane.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `deploy/cloud/docker-compose.yml` `mc-server` CPU & memory/heap defaults | #55 / #73 non-heap headroom contract + ADR-0019 | `NEO_MC_SERVER_HEAP_MB`, `NEO_MC_SERVER_MEMORY_LIMIT`, and `cpus` sized from post-#1843 13-seat telemetry with non-heap margin intact | Existing operator `.env` overrides remain honored | Inline arithmetic comment in `docker-compose.yml` (per #60/#73 bar) | `DeclaredHeapCeilings.spec.mjs` + 24 h `[L3, post-merge]` telemetry |

Decision Record impact: aligned-with ADR-0019 and ADR-0026.

## Acceptance Criteria

- [ ] Post-#1843 `mc-server` heap peak, non-heap RSS, and CPU throttling measurements are recorded in the PR body / inline compose comment, and `deploy/cloud/docker-compose.yml` updates `NEO_MC_SERVER_HEAP_MB`, `NEO_MC_SERVER_MEMORY_LIMIT`, and `cpus` while passing `DeclaredHeapCeilings.spec.mjs`.
- [ ] `[L3, post-merge]` After plane rebuild, `mc-server` runs 24 h under normal fleet load without exit `139`/`137` OOM and without >10 s `who_is_online` stalls tripping FM's `Roster read timed out` banner.

## Out of Scope

- `mc-server` SQLite/JSON query optimization and event-loop stall attribution (#1843).
- Chroma `store-ceiling-at-cap` diagnosis routing.

## Avoided Traps

- Raising `NEO_MC_SERVER_HEAP_MB` without simultaneously raising `NEO_MC_SERVER_MEMORY_LIMIT` (#55: trades catchable V8 OOM exit `139` for uncatchable kernel `SIGKILL` exit `137`).
- Sizing caps from pre-#1843 query-scan spikes before #1843 lands.

## Related

Blocked by #1843 · Related: #55, #60, #73, #1416

---

Live latest-open + closed sweep: checked latest 20 open of 154 (`red` band; mechanical split from #1843) + open/closed searches (`mc-server headroom`, `#55`, `#73`; Out of Scope & Avoided Traps read on #60) at 2026-11-25T16:59Z; no duplicate open ticket.
A2A in-flight sweep: latest 30 all-status messages read at 2026-11-25T17:01Z; split requested by `@neo-opus-ada` (`MESSAGE:0f1a6905-9be4-485b-ae33-f0aa031583c5`).
MC sweep: `"chroma store-ceiling-exhaustion raise-ceiling executor-failed 16 GiB cap mc-server heap cpu"`, 5 results, no duplicate post-#1843 sizing ticket.
Own-assignment sweep: 5 open, none overlapping.
Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): `deploy/cloud/docker-compose.yml` + `test/playwright/unit/ai/daemons/orchestrator/DeclaredHeapCeilings.spec.mjs`.

unowned-rationale: Blocked on #1843 hot-query optimization + live measurement; curated contributor labels omitted because `[L3, post-merge]` requires live maintainer plane rebuild.

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "mc-server NEO_MC_SERVER_HEAP_MB NEO_MC_SERVER_MEMORY_LIMIT cpus throttling DeclaredHeapCeilings post-1843"

## Timeline

- 2026-11-25T17:03:53Z @neo-gemini-julia added the `enhancement` label
- 2026-11-25T17:03:53Z @neo-gemini-julia added the `ai` label
- 2026-11-25T17:03:53Z @neo-gemini-julia added the `performance` label
- 2026-11-25T17:03:53Z @neo-gemini-julia added the `agent-os` label
- 2026-11-25T17:04:43Z @neo-gemini-julia marked this issue as being blocked by #1843
- 2026-11-25T17:04:50Z @neo-gemini-julia cross-referenced by #1843
- 2026-11-25T17:15:46Z @neo-opus-ada cross-referenced by #1931
- 2026-11-25T17:42:35Z @neo-opus-ada cross-referenced by #1933
- 2026-11-25T19:09:06Z @neo-opus-ada cross-referenced by #1939
- 2026-11-25T19:09:13Z @neo-opus-ada cross-referenced by #1940

