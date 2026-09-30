---
id: 579
title: 'The container probe times out on its own 8 s default under a 15 s compose timeout, so the 180 s detection bound #568 states assumes a check the probe never runs'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T13:16:30Z'
updatedAt: '2026-09-30T22:32:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/579'
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
# The container probe times out on its own 8 s default under a 15 s compose timeout, so the 180 s detection bound #568 states assumes a check the probe never runs

## Context

mc-server on the `46ab45f` cut (2026-09-27), `docker inspect .State.Health.Log` at 13:03:26Z: `exit=1 :: MCP healthcheck tool call timed out after 8000ms — [service-unresponsive] this probe was ready after 333ms, well inside its 8000ms budget, and then tool call still produced nothing.` The next probe (13:04:05Z) answered in 201 ms startup; `FailingStreak` reset from 1 to 0. The probe's compose declaration reads `timeout: 15s` (#568 / PR #570), but the check ended at 8 s. Defect-note fingerprint `e88cebb7432f2b73` (13:13Z).

## The Problem

`ai/scripts/diagnostics/mcpHealthcheck.mjs` bounds its connect and tool-call operations by `--timeout-ms` (default `DEFAULT_TIMEOUT_MS = 8000`, or `NEO_MCP_HEALTHCHECK_TIMEOUT_MS`). No compose file passes it. `deploy/cloud/docker-compose.yml` (kb-server `:214`, mc-server `:388`), the local overlay (`docker-compose.local-agent-os.yml:50, :90`) and the dev parity file (`docker-compose.dev.yml:253, :323`, whose own cadence is 15 s / 10 s / 10) pass `--url`, `--client-name` and `--expected-*` only. So a `healthcheck` tool call slower than 8 s fails the probe at 8 s, and the compose `timeout: 15s` never applies. #568's detection bound — `retries × (interval + timeout)` = 4 × (30 + 15) = 180 s — and the R1 rationale that the 15 s "covers a contention-inflated interpreter start" describe a check the probe does not run; the effective worst case is 4 × (30 + 8) = 152 s, and the effective tolerance for a slow tool call is 8 s, not 15. On this plane the tool call exceeds 8 s under REM load (24 h: avg 4.2 s, max 366 s; 46ab45f's first two hours: avg 2.33 s, max 126 s), so four consecutive slow calls would mark the container `unhealthy` at a threshold #570's numbers do not describe.

## The Architectural Reality

- `ai/scripts/diagnostics/mcpHealthcheck.mjs:23` `DEFAULT_TIMEOUT_MS`, `:194` the `--timeout-ms` option (env `NEO_MCP_HEALTHCHECK_TIMEOUT_MS`), `:26–78` `classifyProbeFailure` (probe-starved vs service-unresponsive, keyed on `startupMs >= timeoutMs`).
- `deploy/cloud/docker-compose.yml` the two MCP-probed `healthcheck` blocks (`interval 30s / timeout 15s / retries 4`), the local overlay's `test` overrides.
- `test/playwright/unit/ai/deploy/DeclaredHeapCeilings.spec.mjs` "MCP container probes: cadence" pins the compose values; `test/playwright/unit/ai/daemons/orchestrator/services/DeploymentStateBridgeService.spec.mjs` "the direct probe outlives the container healthcheck it second-guesses (cross-artifact)" reads the compose `timeout` — both read the compose number, neither the probe's own budget.
- Structure map: `deploy/cloud` (compose) and `ai/scripts/diagnostics` (the probe); no new file.

## The Fix

1. The probe's budget is its block's timeout less one second. Every probe command passes `--timeout-ms`: 14000 under the base and local overlay's 15 s, and 9000 under the dev parity file's 10 s. The budget bounds each operation, the connect and then the tool call, while Docker's timeout bounds the whole check. After the usual sub-second connect, the probe's own timeout fires first with its timing split; after a slow connect, Docker can end the check first.
2. The probe spec's every-compose-file sweep pins the relation for each `mcpHealthcheck.mjs` block in every file. A second arm records the deadlines the probe arms when each real command drives it.
3. #568's comments and #64 AC-9 name the effective tolerance (14 s) where they said 15 s.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| compose probe commands (kb-server, mc-server; base, local overlay, dev parity) | `docker-compose*.yml` | `--timeout-ms` at the block's timeout less one second (14000; 9000 in parity) | the script default (8 s) when absent, which is what ships today | compose comment | the every-file budget arm |
| the probe's budget | `mcpHealthcheck.mjs` | unchanged option; the caller sets it | `DEFAULT_TIMEOUT_MS` | script JSDoc | the deadline-receipt arm |
| detection bound | #568 | 4 × (30 + 15) = 180 s with the probe's 14 s inside | — | compose comment | the every-file budget arm |

## Acceptance Criteria

- [ ] AC-1: every `mcpHealthcheck.mjs` probe command in the deploy compose files (base, local overlay, dev parity) carries `--timeout-ms` one second inside the timeout that ends its check, pinned over every file (red on today's commands).
- [ ] AC-2: each compose probe command arms its connect and tool-call deadlines at that budget, so a tool call that answers after 10 s passes when startup and the connect leave it that much of Docker's 15 s (unit: `runHealthcheck` driven by each real compose command, recording the deadlines it arms; red at the 8 s default).
- [ ] AC-3 (post-merge, L4; receipt on #64): on the recreated plane, `docker inspect` shows the probe command with the budget, and one hour's health log carries no 8 s self-timeout.

## Out of Scope

- The healthcheck tool's own per-call cost (the wrapper's composition under REM load) — the lane that makes slow calls rare rather than tolerated.
- The direct-probe budget (`directProbeTimeoutMs`) — its cross-artifact arm keeps reading the compose timeout.

## Related

#568, PR #570, #64, neomjs/neo#16646.

Live latest-open sweep: latest 20 open Brain issues at 2026-09-27T13:12Z hold no equivalent; exact sweeps `mcpHealthcheck timeout` / `NEO_MCP_HEALTHCHECK_TIMEOUT_MS` → none. A2A in-flight sweep (last 30, any read state): none on the probe. MC sweep: Grace's 08-07/08-09 probe-deadline turns (a downstream plane's tuning) and today's #568 turns; the 8 s self-timeout under the 15 s compose bound was not seen before. Own-assignment sweep: 8 open, none overlapping. Structure map: `deploy/cloud`, `ai/scripts/diagnostics`; no new file.

Decision Record impact: none.

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a
Retrieval Hint: "mcpHealthcheck.mjs DEFAULT_TIMEOUT_MS 8000 --timeout-ms under compose timeout 15s; probe tool call timed out after 8000ms while service answered next probe; #568 detection bound"


## Timeline

- 2026-09-27T13:16:30Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T13:16:32Z @neo-opus-vega added the `bug` label
- 2026-09-27T13:16:32Z @neo-opus-vega added the `ai` label
- 2026-09-27T13:16:32Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T13:24:01Z @neo-opus-vega cross-referenced by #64
- 2026-09-27T17:09:14Z @neo-opus-vega cross-referenced by #593
- 2026-09-27T17:19:43Z @neo-opus-vega cross-referenced by PR #594
- 2026-09-30T22:11:30Z @neo-opus-vega cross-referenced by PR #655
- 2026-09-30T22:31:46Z @neo-opus-vega referenced in commit `e528e00` - "fix(deploy): the probe budget is per operation, and CI executes its arms (#579)

`--timeout-ms` bounds each of the probe's operations, the connect and then the tool
call; Docker's timeout bounds the whole check. The compose rationale and the arms'
wording now say that, instead of promising that the probe always reports before
Docker ends the check: after a slow connect, Docker can end it first. The Brain Unit
smoke step now runs mcpHealthcheck.spec.mjs. It had only collected the spec, so the
two new arms never executed in CI."

