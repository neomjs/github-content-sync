---
id: 557
title: 'healthcheck''s top-level status folds maintenance posture in, so a serving Memory Core reports degraded'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-26T20:29:32Z'
updatedAt: '2026-09-26T21:38:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/557'
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
closedAt: '2026-09-26T21:38:47Z'
---
# healthcheck's top-level status folds maintenance posture in, so a serving Memory Core reports degraded

## Context

Operator ruling, 2026-09-26 (repeated; raised before): *the `degraded` flag is nonsense — it means heavy maintenance is running and not caught up, but MC itself is healthy and should say so; there could be another flag for the other items.*

Live sample the same evening (mc-server at Brain dev `b0b85e8`, 19:56Z, 15 s after boot): `status: "degraded"`, `details: ["Connected to the orchestrator-managed ChromaDB instance", "Backup maintenance is degraded: off-host-durability-unmet, backup-retry-exhausted, backup-state-conflict."]`, while `runtimeFreshness: current`, `database.connection.connected: true` (41,734 memories), `memoryWalDrain: caught-up`, `corpusProjectionFreshness: healthy`, `heavyMaintenanceStarvation: consumed-clear`, `serviceMemoryPressure: consumed-clear`. The server was serving every tool. Eos read the same payload during the 20:2xZ incident as "serving-but-degraded" and had to say so in words, which is the tell: the flag does not mean what its consumers need it to mean.

#211 (closed 2026-08-30) met the same symptom on the consumer side only: the `self-repair` skill stopped triggering on the top-level flag (skills #20); its closing comment states "No Brain health field changed". The producer is unchanged.

## The Problem

`HealthService` composes advisory maintenance posture into the serving verdict. Three sites turn a `healthy` payload into `degraded`:

- `HealthService.mjs:1125–1126` — a service memory-pressure receipt;
- `:1205–1206` — a heavy-maintenance starvation receipt with `posture: 'degraded'`;
- `:1927–1932` — the embedding canary `failed | terminal | stale`;

and the backup-maintenance observation adds its line to `details` under the same flag (today's sample). Of these only the canary is about serving (semantic recall depends on it); the rest are about maintenance lanes the operator can read in their own axes, which the payload already carries (`maintenance.backup.status`, `heavyMaintenanceStarvation.posture`, `serviceMemoryPressure.state`).

## The Architectural Reality

- The payload is already two-layered in fact: serving facts (`runtimeFreshness`, `database.connection`, `memoryWalDrain`, `providers`, the canary) and maintenance posture (`maintenance.*`, `heavyMaintenanceStarvation`, `serviceMemoryPressure`, `corpusProjectionFreshness`, `backup`). Only the top-level `status` mixes them.
- The docker HEALTHCHECK accepts `healthy,degraded`, so the container verdict is unaffected either way.
- `who_is_online`, the fleet's plane card and every fresh agent session read `status` first; ADR 0025/0026's self-heal posture wants a flag that means "act" only when serving is impaired.

## The Fix

1. `status` reflects serving health only: transport ready, database connected, WAL drain not stalled, providers reachable, canary not failed. `unhealthy` when the server cannot serve, `degraded` when it serves with impaired recall (canary), `healthy` otherwise.
2. A top-level `posture` (`clear | attention`) plus an `advisories[]` list carry the maintenance items that used to flip the flag (backup durability, heavy-maintenance starvation, service memory pressure, corpus projection staleness), each with its reason codes, unchanged from the axes they come from.
3. `details` keeps "All features are operational" only when both `status` is `healthy` and `advisories` is empty, and otherwise names the advisories, so the prose and the flags agree.

## Contract Ledger Matrix

Consumed fields of the `healthcheck` payload (Memory Core MCP tool, the docker HEALTHCHECK, the container-health controllers, the fleet's plane card). Added 2026-09-26 at the reviewer's request on PR #559.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `status` | `HealthService#performHealthCheck` + `composeMemoryCoreHealthcheck` | Serving verdict only: `unhealthy` (database unreachable, startup dependency not ready), `degraded` (embedding canary failed/terminal/stale, WAL embed drain stalled), `healthy` otherwise; maintenance posture never moves it | unchanged for every serving impairment; an `unhealthy` verdict always wins | `openapi.yaml` `HealthCheckResponse.status` description | canary and dependency arms (`HealthService.spec.mjs`), the drain arm (`HealthcheckBackupDetails.spec.mjs`) |
| `posture` | `noteHealthAdvisory` (HealthService) | `clear` when `advisories` is empty, `attention` when at least one advisory is open; set by every advisory producer, idempotent | `clear` on the base payload and on a cached payload before the folds run | `HealthCheckResponse.posture` | fold specs, `HealthcheckBackupDetails.spec.mjs`, `McpServerToolLimits.spec.mjs` |
| `advisories[]` | `noteHealthAdvisory` | One entry per open axis: `{axis, state, …}` with the axis's own reason codes / breaches / atCap / modes; axes: `backup`, `heavyMaintenanceStarvation`, `serviceMemoryPressure`, `corpusProjectionFreshness`, `providerAdmission` (the documented scope delta: identities served from the validation cache while the provider cannot be asked) | `[]`; a stale, unknown or unavailable inspection never produces an advisory except starvation `consumed-unknown`, which is recorded as one | `HealthCheckResponse.advisories` (items `required: [axis, state]`) | the same specs; the #16060 schema-completeness arm keeps producer keys and schema in step |
| retained axis sections (`maintenance.backup`, `heavyMaintenanceStarvation`, `serviceMemoryPressure`, `corpusProjectionFreshness`) | their producers, unchanged | unchanged; the advisory duplicates their verdict, never replaces it | unchanged | existing descriptions | existing arms unchanged |
| `details[]` | every producer | the same prose lines as before; "All features are operational" only when `status` is `healthy` and no advisory is open | unchanged | existing description | the withdrawal arms in both fold specs |
| failure behaviour | fold sites | a fold that throws is recorded as `fold-error` on its axis section, as today; no advisory is fabricated from a failed read | unchanged | — | existing `fold-error` arms |

## Acceptance Criteria

- [ ] AC-1: a payload with a connected database, a caught-up WAL drain, reachable providers and a passing canary reports `status: 'healthy'` even with a degraded backup receipt, a starving heavy-maintenance lane and a memory-pressure receipt present; those appear under `posture: 'attention'` and `advisories[]` with their reason codes (unit arms on `HealthService`, red-first on the three sites above).
- [ ] AC-2: a failed or stale canary and a stalled WAL embed drain still report `degraded` (serving impaired, not stopped), and an unreachable database or an unready startup dependency reports `unhealthy`, as today (existing arms kept green; corrected 2026-09-26: the first draft named the drain under `unhealthy`).
- [ ] AC-3: the openapi / tool description of `healthcheck` documents `status`, `posture` and `advisories` with the meaning above, and the `self-repair` skill's trigger text is checked against it (consumer side unchanged if it already keys on the axes, per #211).

## Out of Scope

- The cost of the health probe itself under the cockpit's tick (defect-noted 2026-09-26 20:03Z, fingerprint `024c440c69570d32`).
- The `who_is_online` presence semantics (#31).

## Decision Record impact

aligned-with ADR 0025 / ADR 0026 (self-diagnosis and self-heal act on serving impairment, not on advisory posture).

## Related

#211 (the consumer half, closed) · neo-agent-skills#20 · #64 (today's plane receipts)

Live latest-open sweep: checked the latest 20 open Brain issues at 20:28Z; no equivalent. Historical sweep: `gh issue list --search "healthcheck degraded status"` over open and closed — #211 (consumer side) and #233 (backup alarm guard) are the neighbours, neither changes the producer's verdict. A2A claim sweep: the 20:0xZ–20:28Z mailbox carries no claim on this scope. Memory sweep: `query_raw_memories` timed out (the server was restarting). Own-assignment sweep: none of my open tickets covers it.

Origin Session ID: 27467eea-851e-486b-a0ca-55f744b67fdf
Retrieval Hint: "healthcheck top-level status degraded backup maintenance posture advisories serving health operator ruling"


## Timeline

- 2026-09-26T20:29:32Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-26T20:29:33Z @neo-opus-vega added the `bug` label
- 2026-09-26T20:29:33Z @neo-opus-vega added the `ai` label
- 2026-09-26T20:32:47Z @neo-preview cross-referenced by PR #556
- 2026-09-26T20:35:29Z @neo-opus-vega cross-referenced by #459
- 2026-09-26T21:07:34Z @neo-opus-vega cross-referenced by PR #559
- 2026-09-26T21:27:10Z @neo-opus-vega referenced in commit `ee76113` - "docs(memory-core): the health composition's descriptions say what the code does — advisories raise posture, status stays the serving verdict (#557)"
- 2026-09-26T21:38:47Z @tobiu referenced in commit `c6c92c2` - "Merge pull request #559 from neomjs/vega/557-health-status-serving-only

fix(memory-core): healthcheck's status is the serving verdict; maintenance posture rides posture and advisories (#557)"
- 2026-09-26T21:38:47Z @tobiu closed this issue
- 2026-09-26T21:42:58Z @neo-opus-vega cross-referenced by #64

