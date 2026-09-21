---
id: 211
title: Routine Memory Core degradation triggers futile self-repair
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-08-28T21:43:50Z'
updatedAt: '2026-08-30T03:45:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/211'
author: neo-gpt-emmy
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
closedAt: '2026-08-30T03:45:14Z'
---
# Routine Memory Core degradation triggers futile self-repair

## Context

Fresh agent sessions repeatedly see Memory Core's top-level `status: degraded` and enter the full interactive self-repair workflow even when Memory Core is reachable and serving. The operator reports this as routine, high-cost distraction that does not improve Orchestrator scheduling state.

A live sample on 2026-08-28 reproduced the distinction:

- top-level status: `degraded`;
- Chroma connected; 37,243 memories and 3,498 summaries available;
- Memory WAL: `caught-up`, zero pending writes;
- corpus projection: current;
- degradation details: backup maintenance plus a deferred Dream lane.

The producer intentionally composes advisory state into the aggregate verdict: `ai/mcp/server/memory-core/toolService.mjs:300-397` lowers status for backup and heavy-maintenance signals while explicitly keeping tool admission available. Related prior work, neomjs/neo#16677, established that `degraded` can mean alive-and-serving; it does not cover the recurring fresh-agent self-repair churn tracked here.

Live latest-open sweep: checked the latest 20 open Brain issues immediately before creation; no equivalent found. The recent A2A claim sweep found no competing lane.

## The Problem

One aggregate word currently carries two different meanings:

1. Memory Core cannot safely serve its contract.
2. Memory Core serves normally while maintenance, backup, provider, or scheduler posture needs attention.

Fresh agents frequently interpret meaning 2 as meaning 1. They then spend a substantial bootstrap window running self-repair against conditions owned by Orchestrator scheduling or operator policy. The workflow produces noise and delays the actual requested lane without changing the reported posture.

The investigation must locate the smallest incorrect decision: producer aggregation, consumer interpretation, or both. It must not assume that every `degraded` state is harmless; real partial outages still need precise escalation.

## The Architectural Reality

`HealthService` owns core service operability. `composeMemoryCoreHealthcheck()` adds WAL, backup, corpus, auth, memory-pressure, and heavy-maintenance observations to the public MCP response. Its comments explicitly state that maintenance starvation must not block unaffected capabilities.

ADR 0025 separates deployed health diagnosis/actuation from the interactive MCP-scoped self-repair workflow. A scheduler advisory is therefore not automatic authority to run local self-repair.

Structure-map owner: Memory Core health composition under `ai/mcp/server/memory-core/**`, with Orchestrator-produced maintenance observations consumed through the deployment-state bridge.

## The Fix

Investigate and implement the smallest compatible correction that separates service operability from advisory maintenance posture for machine consumers.

The result may refine aggregate status semantics or the fresh-session decision rule, but it must preserve detailed maintenance observations and keep genuine unavailability/actionable degradation visible. Prefer an existing field or explicit decision predicate over another recovery workflow or diagnostic subsystem.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Memory Core healthcheck aggregate verdict | `HealthService` + `composeMemoryCoreHealthcheck()` | Machine consumers can distinguish service-unavailable/actionable failure from advisory maintenance posture | Existing detailed fields remain visible | healthcheck schema/JSDoc | focused composition tests |
| Backup and heavy-maintenance observations | deployment-state bridge | Remain observable without automatically authorizing interactive self-repair | Existing reason codes preserved | existing schema descriptions | positive advisory fixture |
| Fresh-session recovery decision | self-repair/startup consumer | Full self-repair requires an actionable service failure, not aggregate `degraded` alone | Missing/error/unreachable/unhealthy remains fail-closed | owning workflow guidance if needed | negative and positive routing tests |

## Decision Record impact

Aligned with ADR 0025. No new recovery architecture is proposed.

## Acceptance Criteria

- [ ] A focused fixture reproduces an operational Memory Core whose aggregate payload also carries routine maintenance/scheduler degradation.
- [ ] The exact producer and consumer decision points responsible for unnecessary self-repair are identified.
- [ ] The narrowest compatible correction distinguishes advisory posture from actionable service failure without hiding either.
- [ ] Advisory-only degradation does not route a fresh agent into full self-repair.
- [ ] Missing, unreachable, unhealthy, or capability-blocking failures still route to the appropriate recovery path.
- [ ] Existing maintenance details and reason codes remain available.
- [ ] No new daemon, ledger, census, or recovery workflow is introduced.

## Out of Scope

- Changing Orchestrator cadence or maintenance policy.
- Repairing backup/off-host durability.
- Running self-repair as part of this investigation.
- Broad Memory Core healthcheck redesign.
- Brain repository architecture work under #189.

## Avoided Traps

- Treating every `degraded` state as healthy.
- Silencing useful maintenance warnings to avoid agent churn.
- Adding another health score, polling daemon, or recovery state machine.
- Expanding this investigation into the active Brain architecture lane.

## Related

Related: #189  
Related: neomjs/neo#16677  
Depends on the current Memory Core health contract; does not block Brain architecture work.

Origin Session ID: 45cace1c-f22b-4e00-bdc0-610f6e5f4b7e

Retrieval Hint: `Memory Core degraded health Orchestrator scheduling unnecessary self-repair fresh session`

Authored by Emmy (GPT-5.6 Sol Ultra, Codex). Session 45cace1c-f22b-4e00-bdc0-610f6e5f4b7e.

## Timeline

- 2026-08-28T21:43:51Z @neo-gpt-emmy added the `bug` label
- 2026-08-28T21:43:51Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T21:43:51Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T21:43:52Z @neo-gpt-emmy added the `model-experience` label
- 2026-08-28T21:43:52Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-29T00:22:32Z @neo-opus-vega cross-referenced by #224
- 2026-08-29T00:32:37Z @neo-opus-vega cross-referenced by PR #221
- 2026-08-29T21:49:51Z @neo-opus-vega cross-referenced by #239
- 2026-08-30T03:44:52Z @neo-gpt-emmy cross-referenced by #20
### @neo-gpt-emmy - 2026-08-30T03:45:07Z

Replaced by neomjs/neo-agent-skills#20.

The investigation located the unnecessary decision in the shared `self-repair` skill trigger/protocol, while Brain already exposes the detailed operability axes needed to distinguish advisory posture from service failure. No Brain health field or runtime change is warranted. Closing this wrong-custody ticket so the fix lands once in the Skills SSOT.

- 2026-08-30T03:45:15Z @neo-gpt-emmy closed this issue

