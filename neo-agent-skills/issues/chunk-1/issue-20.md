---
id: 20
title: Self-repair requires an actionable service failure
state: OPEN
labels:
  - bug
  - ai
  - testing
  - model-experience
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-30T03:44:51Z'
updatedAt: '2026-08-30T04:11:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/20'
author: neo-gpt-emmy
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
# Self-repair requires an actionable service failure

## Context

Brain issue neomjs/neo-agent-brain#211 measured a recurring startup failure mode: Memory Core is reachable, serving 37k+ memories, WAL caught up, and corpus projection current, while its aggregate healthcheck still reports `degraded` because backup or heavy-maintenance posture needs attention. Fresh agents interpret that aggregate word as service failure and enter the full self-repair workflow, which changes none of the advisory state and delays their requested lane.

The ownership check now identifies the consumer: `.agents/skills/self-repair/SKILL.md` includes bare `system degraded` in its trigger vocabulary, and the Phase-0 protocol distinguishes attachment/service/authority but never distinguishes an advisory aggregate from an actionable service failure. Brain already exposes the detailed axes needed for that decision.

Live latest-open sweep: checked the latest 20 open Skills issues immediately before creation; no equivalent exists. A2A claim sweep over the latest 30 messages found no overlapping claim.

## The Problem

One trigger currently collapses two states:

1. Memory Core is missing, unreachable, unhealthy, or unable to serve the required capability.
2. Memory Core serves normally while backup, provider, corpus, or scheduler observations lower the aggregate verdict.

The router's bare `system degraded` phrase makes state 2 sufficient to invoke an expensive recovery workflow. The protocol then starts infrastructure verification without first testing whether the aggregate degradation is actionable for the capability the agent needs.

Changing Brain's producer would be the wrong boundary: the maintenance details are truthful and valuable, and adding a second health score would duplicate the existing operability axes. The decision belongs to the agent-consumed self-repair rule.

## The Architectural Reality

- `.agents/skills/self-repair/SKILL.md` is the always-loaded trigger Map. It must stay a compact router.
- `.agents/skills/self-repair/references/self-repair-protocol.md` is the conditional Atlas and already owns Phase-0 failure classification.
- Brain's Memory Core healthcheck already reports database connectivity, WAL state, corpus freshness, runtime freshness, backup posture, and heavy-maintenance posture separately. The skill consumes those facts; it does not need a new producer field.
- ADR 0025 separates deployed health diagnosis/actuation from interactive MCP-scoped self-repair. Scheduler advisory state is not automatic authority for interactive repair.

Structure-map gate: the runtime producer remains under `ai/services/memory-core/**`; this ticket changes only the shared skill consumer in the Skills SSOT.

## The Fix

- Narrow the router trigger so a bare aggregate `degraded` result is not by itself sufficient. Explicit healthcheck/diagnostic requests and actionable service failures still trigger.
- Add one Phase-0 decision table using existing fields:
  - missing tools, failed attachment, unreachable service, disconnected core storage, stalled required write path, or capability-blocking failure ⇒ self-repair/diagnosis;
  - reachable and serving, with degradation attributable only to advisory maintenance/scheduler posture ⇒ retain the advisory and continue the requested lane.
- Keep missing/ambiguous evidence fail-closed. The rule must not relabel every degraded state healthy.
- Add no daemon, health score, recovery state machine, Brain field, or polling loop.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `self-repair/SKILL.md` trigger | skill frontmatter | explicit health tasks or actionable failure trigger; bare aggregate `degraded` does not | missing/erroring required tools still trigger | router only | manifest/frontmatter lint |
| Phase-0 routing predicate | self-repair protocol + existing healthcheck axes | distinguish capability-blocking failure from advisory-only posture before Phase 1 | missing or ambiguous operability evidence remains fail-closed | protocol decision table | positive/negative fixture matrix |
| Brain health payload | Memory Core healthcheck | unchanged; detailed maintenance observations remain visible | n/a | no Brain docs change | zero Brain diff |

## Decision Record impact

Aligned with ADR 0025. No amendment required.

## Acceptance Criteria

- [ ] An operational Memory Core with advisory-only backup/heavy-maintenance degradation does not route a fresh agent into full self-repair.
- [ ] Missing tools, attachment failure, unreachable service, disconnected core storage, stalled required writes, and capability-blocking failures still route to diagnosis/recovery.
- [ ] Missing or ambiguous operability evidence remains fail-closed.
- [ ] Backup, heavy-maintenance, and other advisory details remain visible and are not renamed healthy.
- [ ] The `SKILL.md` delta stays a trigger-only Map change; decision detail lives in the existing Phase-0 Atlas.
- [ ] Skill manifest/frontmatter and payload-budget checks pass.
- [ ] No Brain runtime, health schema, daemon, ledger, score, or polling workflow is added.

## Out of Scope

- Changing Memory Core aggregation or Orchestrator maintenance policy.
- Repairing backup/off-host durability.
- Running self-repair during implementation.
- Broad healthcheck redesign.

## Avoided Traps

- Treating every degraded state as healthy.
- Silencing truthful maintenance warnings.
- Adding a second aggregate score instead of interpreting existing axes.
- Duplicating the decision in consumer repos instead of the Skills SSOT.

## Related

Replaces the wrong-custody Brain ticket neomjs/neo-agent-brain#211. Related: neomjs/neo#16677.

Origin Session ID: 5c37632a-b342-4f84-a66f-61510b8382d5

Retrieval Hint: `query_raw_memories("Memory Core degraded advisory self-repair fresh session")`

## Timeline

- 2026-08-30T03:44:53Z @neo-gpt-emmy added the `bug` label
- 2026-08-30T03:44:53Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T03:44:53Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T03:44:53Z @neo-gpt-emmy added the `model-experience` label
- 2026-08-30T03:44:53Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-30T03:45:08Z @neo-gpt-emmy cross-referenced by #211
- 2026-08-30T04:11:28Z @neo-gpt-emmy assigned to @neo-gpt-emmy

