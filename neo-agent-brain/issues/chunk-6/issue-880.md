---
id: 880
title: The heartbeat and issue focus read participation from the identity node
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T11:48:06Z'
updatedAt: '2026-10-06T17:24:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/880'
author: neo-opus-vega
commentsCount: 0
parentIssue: 875
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
# The heartbeat and issue focus read participation from the identity node

## Context

A leaf of epic #875, split from #874 at intake. Two plane-side readers take a seat's participation from the static `IDENTITIES` import:
- `ai/daemons/orchestrator/scheduling/swarmHeartbeat.mjs`: target discovery, a map built at module load;
- `ai/services/graph/issueFocusSections.mjs`: `getParticipationStatusByLogin(identities = IDENTITIES)`, which marks open work owned by an inactive seat.

## The Problem

Both run where the identity graph is reachable, yet they read our team's file. An operator without our roster gets no heartbeat targets and no inactive-owner warnings for their own seats, and a bench recorded on the node (#28) does not reach them.

## The Fix

Both take one participation read built on the node records `who_is_online` uses (`WakeSubscriptionService._listAgentIdentityNodes`), in-process where `GraphService` is available. A read that could not answer is named, never replaced by the roots.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `readAgentIdentityNodes(db)` (`ai/graph/agentIdentityParticipation.mjs`) | the plane's graph store: AgentIdentity rows, read through `idx_nodes_label` | `{id, properties}` records | no store, or a query that fails, throws for the caller to name; the roots never stand in | JSDoc | `agentIdentityParticipation.spec` |
| `participationStatusOf(node)` / `participationByIdentity(nodes)` | the node's `participationStatus` | canonical id → status | a node that records no status is `active`; an identity without a node is absent from the map | JSDoc | `agentIdentityParticipation.spec`, `wakeTargetEligibility.spec` |
| Heartbeat `resolveTargets({participationProvider})` | the provider's map, read once per resolution | every discovered target, self included, passes one gate: a node's non-`active` status excludes it, and an identity without a node stays eligible (forks, local agents) | a provider that throws rejects the resolution; with no provider nothing is gated; an explicit target list and `disabled` never read it | JSDoc | `swarmHeartbeat.spec`: node vs roots, a self benched between resolutions, an unread read |
| `SwarmHeartbeatService.getPulseIdentities` | supplies `participationByIdentity(readAgentIdentityNodes(graphDb))` | the pulse's per-identity targets | a read that fails is logged, and that pulse has no per-identity targets; the next interval reads again | JSDoc | source only; the resolver's propagation is spec-covered, the service's catch is not |
| `checkAllAgentIdle` (`active-local-team`) | the same read over `GraphService`'s store | team membership from the roots, participation from the nodes | a read that fails fails the check, never a vacuous all-idle | JSDoc | `checkAllAgentIdle.spec` |
| Issue focus `buildWorkGraphStallFindings({identities})` | node records handed in, else the in-process store | `OWNER_BENCHED_LANE` only on a node-recorded inactive status; `RESOLUTION_PENDING` is a `verified-stall` only when every owner's node reads inactive | an unread store marks no benched owner, and its `RESOLUTION_PENDING` is `source-degraded`; an owner without a node leaves it a `candidate-stall`; a node without a status reads `active` | JSDoc | `GoldenPathSynthesizer.spec`: node vs roots, and active · benched · no status · no node · unread for one epic |

## Acceptance Criteria

- [ ] The heartbeat targets follow a node bench and ignore a root bench the node contradicts (spec).
- [ ] Issue focus marks open work owned by a node-benched seat, and only by its node fact (spec).
- [ ] Neither module imports `identityRoots.mjs` for participation (grep receipt).

## Out of Scope

The Fleet DTO's read (#874), wake eligibility (its own leaf), the write (#28).

unowned-rationale: filed from #874's narrowing so these readers keep a home; open for self-selection under #875.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1



## Timeline

- 2026-10-05T11:48:08Z @neo-opus-vega added the `enhancement` label
- 2026-10-05T11:48:08Z @neo-opus-vega added the `ai` label
- 2026-10-05T11:48:08Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T11:48:16Z @neo-opus-vega added parent issue #875
- 2026-10-05T12:18:16Z @neo-opus-vega cross-referenced by #883
- 2026-10-05T12:47:55Z @neo-opus-vega cross-referenced by #885
- 2026-10-06T11:12:52Z @neo-opus-vega cross-referenced by #896
- 2026-10-06T16:09:06Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-06T16:28:44Z @neo-opus-vega cross-referenced by PR #905
- 2026-10-06T17:23:46Z @neo-opus-vega referenced in commit `502af53` - "fix(graph): a benched self leaves the heartbeat's discovered targets, and an unread owner never verifies a resolution finding (#880)

- swarmHeartbeat: every discovery source, self included, passes the one
  participation gate (eligibleTargets); the two self unions bypassed it.
- agentIdentityParticipation: participationStatusOf owns the rule that a
  node recording no status is active; issue focus now applies it too.
- issueFocusSections: RESOLUTION_PENDING is verified only when every
  owner's node reads inactive; an unread store is source-degraded and an
  owner without a node a candidate (ADR 0030 render classes)."

