---
id: 568
title: The container probe runs the full Memory Core healthcheck every ten seconds
state: CLOSED
labels:
  - enhancement
  - ai
  - performance
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T09:30:35Z'
updatedAt: '2026-09-27T10:56:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/568'
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
closedAt: '2026-09-27T10:56:18Z'
---
# The container probe runs the full Memory Core healthcheck every ten seconds

## Context

Measured on the local plane 2026-09-27 (receipt on #64, comment 5854182787): in the 49 minutes after mc-server's 07:24Z restart, `get_memory_core_tool_metrics` counted `healthcheck` 353 calls (avg 1.07 s, max 31.6 s); the Docker event stream over the same minutes shows the container's own probe (`node ./ai/scripts/diagnostics/mcpHealthcheck.mjs --url http://127.0.0.1:3001 …`) executing every ~11 s, i.e. about 270 of those calls. During the 08:14–08:15Z boot window three probe calls took 63–83 s each. The other callers in that window were the cockpit's brainHealth poll (every 120 s since neomjs/neo-agent-institution#257) and seat turn-starts.

## The Problem

`deploy/cloud/docker-compose.yml` declares the mc-server healthcheck with `interval: 10s`, `timeout: 10s`, `retries: 12` (`:372–376`), and the probe is a full MCP client session that calls the `healthcheck` tool — the same tool a seat calls at turn start: Chroma probe, Docker inspection through the deployment snapshot, corpus projection freshness, wake-subscription delivery reads, WAL drain state. That is a diagnostic payload built for an operator or a seat, run six times a minute by the container runtime to answer one question: is the process serving? On this plane it is the single largest caller of the Memory Core, ahead of every seat and the cockpit together, and each call parses and composes the whole payload on the same event loop that serves the seats.

## The Architectural Reality

- `deploy/cloud/docker-compose.yml:372–376` (mc-server), the sibling declarations for kb-server (`:203–207`) and the orchestrator; `deploy/cloud/docker-compose.local-agent-os.yml:50, :90` override the probe command but not the cadence.
- `ai/scripts/diagnostics/mcpHealthcheck.mjs`: connects a StreamableHTTP MCP client, calls `healthcheck`, checks `--expected-plane-id`, `--expected-plane-data-root`, `--expected-status`. The expectations are the probe's real value: a wrong plane id or data root must fail the container, which a bare TCP check cannot do.
- The runtime's own semantics (Docker's HEALTHCHECK reference): the next `interval` starts after a check completes, and a container is `unhealthy` after `retries` consecutive failures — so a dead service is reported after between `retries × interval` (checks failing at once) and `retries × (interval + timeout)` (every check running to its timeout): 120–240 s at today's `10s / 10s / 12`, 120–180 s at `30s / 15s / 4`. A healthy probe at ~1 s per call runs about 110–120 times an hour at a 30 s interval, not 120 exactly.
- `HealthService#healthcheck({freshObservability})` (mc-server): with `false`, a cached healthy payload younger than five minutes is returned without the Chroma probe and inspection refresh; degraded and unhealthy results are never cached. The MC tool wrapper (`ai/mcp/server/memory-core/toolService.mjs`) still composes drain state, corpus projection freshness, deployment inspection and vector-generation health on every call. kb-server's `healthcheck` tool declares no option (`ai/mcp/server/knowledge-base/openapi.yaml`) and its handler is zero-argument: an extra field is ignored, not rejected (a live call with it succeeded on the plane; PR #570 R1, @neo-gpt).
- Precedent for the class: neomjs/neo#16646 (a probe deadline shorter than the work it waits on trips a healthy service) and the #557 split of serving status from maintenance posture, which is exactly what a container probe should read.

## The Fix

1. `interval: 30s`, `timeout: 15s`, `retries: 4`, `start_period` kept, for mc-server and kb-server in `docker-compose.yml` (the local overlay inherits): a third of the probe calls, and a dead service reported within 120–180 s instead of 120–240 s.
2. The probe passes `freshObservability: false` (the MC tool's existing option): on mc-server, `HealthService` answers from its cached healthy payload inside its five-minute window instead of re-running its Chroma probe and inspection on every tick; the tool wrapper's own state reads are unchanged (out of scope below). On kb-server the field is ignored, so there the cadence is the whole saving. `--expected-status healthy,degraded` keeps reading `status` (the serving verdict since #557), never `posture`.
3. Receipt (L4, on #64 AC-9): one hour on the recreated plane — the probe's execution count from the Docker event stream, the `healthcheck` count and average from `get_memory_core_tool_metrics`, and the container's verdict through a REM run (AC-3).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| kb-server / mc-server compose `healthcheck` (Docker-consumed) | `deploy/cloud/docker-compose.yml` | `interval: 30s`, `timeout: 15s`, `retries: 4`; `start_period` unchanged; detection 120–180 s | the local overlay overrides `test` only and inherits the cadence | compose comments | `DeclaredHeapCeilings.spec` cadence arm: exact values + `retries × (interval + timeout) ≤ 180` |
| probe → mc-server `healthcheck` request | `mcpHealthcheck.mjs`; MC `openapi.yaml` `freshObservability` | `{freshObservability: false}`: cached healthy payload inside HealthService's five-minute window; wrapper reads unchanged | degraded / unhealthy never cached → fresh | script comment | `mcpHealthcheck.spec` call-shape arm; `HealthService.spec` fast-path arms (existing) |
| probe → kb-server `healthcheck` request | KB `openapi.yaml` (no option), zero-argument handler | field ignored; the cadence is the saving | — | script comment | KB `Server.spec` arm: no `freshObservability` advertised; live call receipt (R1) |
| the L4 receipt | #64 AC-9 (ledger row in comment 5854871745) | recorded after the post-merge recreate | — | — | #64 |

## Acceptance Criteria

- [ ] AC-1: the compose declarations for mc-server and kb-server read exactly `interval: 30s`, `timeout: 15s`, `retries: 4`; the cadence arm pins those values and the worst-case bound `retries × (interval + timeout) ≤ 180 s` (red on dev's `10s / 10s / 12`: 240 s); the heap-ceiling arms in the same file stay green.
- [ ] AC-2: the probe's `healthcheck` call carries `freshObservability: false`, pinned by the `mcpHealthcheck.spec` call-shape arm; the KB tool advertises no such input (pinned by a KB `Server.spec` arm) and ignores the field.
- [ ] AC-3 (post-merge, L4; owner #64 AC-9, ledger row in comment 5854871745): one hour on the recreated plane with the cockpit open and the seats active — (a) the Docker event stream shows the mc-server probe executing 100–125 times, each inside its 15 s timeout (about 330 per hour at the 10 s cadence); (b) `get_memory_core_tool_metrics` counts `healthcheck` below 250 for the hour with the same seats and cockpit (the baseline's non-probe callers were ~100 per hour) and its per-call average below the baseline's 1.07 s; (c) no container flips to `unhealthy` through a REM run.

## Out of Scope

- The healthcheck tool's own per-call cost on mc-server — the wrapper's composition of drain state, corpus projection, deployment inspection and vector-generation health (avg 962 ms – 6 s under REM load; defect fingerprint `024c440c69570d32`) — a separate fold-by-fold measurement, as #563 noted.
- The orchestrator's probe (a file stat, not an MCP call).
- Replacing the MCP probe with a TCP check: it would lose the plane-id and data-root expectations the probe exists for.

## Related

#557, #563, #64, #140, neomjs/neo#16646, neomjs/neo-agent-institution#257

Live latest-open sweep: latest 20 open Brain issues at 2026-09-27T09:26Z hold no equivalent; exact sweeps for `healthcheck interval` / `mcpHealthcheck` / `probe every` return #140 (the probe's token gate, a different concern), #373 (closed, the orchestrator's probe), #463/#466 (closed). A2A in-flight sweep (last 30, any read state): none on the probe cadence; the defect-note went out with this morning's all-clear. MC sweep: `query_raw_memories` on the symptom → Grace's 08-07/08-09 probe-deadline turns (the deadline class, a downstream deployment's 60 s / 45 s / 5 / 90 s tuning) and my 09-26 wedge turn; no prior decision on the cadence. Own-assignment sweep: 8 open, none overlapping. Structure map: owning folder `deploy/cloud` (compose) and `ai/scripts/diagnostics` (the probe); no new file.

Decision Record impact: none.

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a
Retrieval Hint: "mc-server docker healthcheck interval 10s mcpHealthcheck.mjs 353 calls per 49 minutes probe load; freshObservability false for the container probe"


## Timeline

- 2026-09-27T09:30:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T09:30:37Z @neo-opus-vega added the `enhancement` label
- 2026-09-27T09:30:37Z @neo-opus-vega added the `ai` label
- 2026-09-27T09:30:37Z @neo-opus-vega added the `performance` label
- 2026-09-27T09:30:38Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T09:44:18Z @neo-opus-vega cross-referenced by PR #570
- 2026-09-27T10:01:17Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T10:14:40Z @neo-opus-vega cross-referenced by PR #569
- 2026-09-27T10:19:54Z @neo-opus-vega referenced in commit `66fd00d` - "feat(deploy): the MCP container probes run every 30 s, and mc-server answers them from its cached healthy payload (#568)"
- 2026-09-27T10:19:54Z @neo-opus-vega referenced in commit `1fb04ae` - "fix(deploy): the probe cadence claims say what Docker does — a third of the calls, 180 s worst-case detection, mc-server's cache leg only (#568)

10 s → 30 s is a third of the probe invocations, not a sixth; Docker starts the
next interval after a check completes and counts consecutive failures, so
detection is bounded by retries × (interval + timeout) = 180 s (240 s before),
not interval × retries. The compose arm now pins the exact 30s / 15s / 4 and
that bound. `freshObservability: false` is honoured by mc-server's HealthService
cache leg only — the MC tool wrapper's own state reads still run, and kb-server
declares no such option (a new KB arm pins the absence). Found on PR #570 R1 by
@neo-gpt."
- 2026-09-27T10:48:20Z @neo-opus-vega referenced in commit `7841a43` - "fix(deploy): the probe blocks' rationale sits above the healthcheck key, inside the cross-artifact arm's window (#568)

`DeploymentStateBridgeService.spec` reads each probe's `timeout` within 400
characters of `mcpHealthcheck.mjs` in the compose text; the R1 comments between
`test` and `timeout` pushed both blocks past that window and the arm found no
probe at all (CI red on 1fb04ae). The comments keep their content above the key."
- 2026-09-27T10:49:47Z @neo-opus-vega cross-referenced by PR #564
- 2026-09-27T10:56:18Z @tobiu referenced in commit `46ab45f` - "Merge pull request #570 from neomjs/vega/568-container-probe-cadence

feat(deploy): the MCP container probes run every 30 s, and mc-server answers them from its cached healthy payload (#568)"
- 2026-09-27T10:56:18Z @tobiu closed this issue
- 2026-09-27T13:16:31Z @neo-opus-vega cross-referenced by #579

