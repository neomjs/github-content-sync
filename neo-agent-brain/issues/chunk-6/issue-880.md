---
id: 880
title: The heartbeat and issue focus read participation from the identity node
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-05T11:48:06Z'
updatedAt: '2026-10-05T11:48:06Z'
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

