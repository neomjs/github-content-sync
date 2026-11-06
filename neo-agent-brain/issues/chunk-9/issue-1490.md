---
id: 1490
title: who_is_online counts identity nodes that record no account type
state: OPEN
labels:
  - bug
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-05T19:12:53Z'
updatedAt: '2026-11-05T19:12:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1490'
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
# who_is_online counts identity nodes that record no account type

## Context

#1485 made four readers count only identity nodes that record an account type, through `ai/graph/agentIdentityParticipation.mjs`. `who_is_online` was not one of them.

On our plane, at dev `5b042434` (2026-11-05 19:09Z, after the cut that deployed #1485), `who_is_online` still lists three unit-fixture nodes as team members: `@agent-a` and `@agent-b` as `dark`, and `@neo-test-writer` as `neverConnected`. Their properties are empty, and #875 (comment 6665339763) traces them to specs that ran against the live graph before unit storage was isolated. The Fleet Manager's presence view renders them as seats.

## The Problem

`WakeSubscriptionService#_listAgentIdentityNodes` selects every node whose label is `AgentIdentity` (`json_extract(data, '$.label') = 'AgentIdentity'`). It applies no membership rule, so a node that records no account type counts as a member.

## The Architectural Reality

- **The membership rule already exists:** `agentIdentityParticipation.mjs`'s `memberLoginsOf()` returns the logins of agents and people, and a node with no `accountType` is not a member.
- **`who_is_online` lists people as well as agents.** `@tobiu` appears as `neverConnected`. So the right predicate is the member rule (agents and people), not the agents-only `agentIdentitiesOf()`.
- **Participation stays orthogonal.** A benched agent keeps its account type and stays in `benched`.

## The Fix

`_listAgentIdentityNodes` drops rows that record no account type, using the predicate behind `memberLoginsOf()`, exported once from `agentIdentityParticipation.mjs` if it is not already. There is no second copy of the rule. A unit arm covers it.

## Acceptance Criteria

- [ ] `who_is_online` leaves out `AgentIdentity` nodes that record no `accountType`. Agents and people stay, and a benched agent stays in `benched`.
- [ ] The membership predicate is the one in `ai/graph/agentIdentityParticipation.mjs`, not a copy.
- [ ] A unit arm seeds an agent, a person, a benched agent and a node with no account type. The last one appears in no bucket.
- [ ] Post-merge, on our plane: `who_is_online` no longer lists `@agent-a`, `@agent-b` or `@neo-test-writer`.

## Out of Scope

- Deleting the fixture nodes from the plane: a data deletion, which needs its own reviewed path.
- Benching the retired `@neo-opus`, an operator decision (#28).
- Admitted Fleet membership (#1297), which may later replace account type as the membership fact.

## Related

#875 (parent) · #1485 (the sibling readers) · #1297

Live latest-open + closed sweep: on 2026-11-05 at 19:1xZ I read the latest 20 open Brain issues and ran the closed searches "who_is_online accountType" and "who_is_online fixture". The hits were #1485, #1400 and #1085, none of them this reader. The A2A sweep of the last 30 messages shows no claim. The Memory Core query returned no prior decision on this symptom. No equivalent found.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The helper exists, and the fix is one filter plus one unit arm.

Retrieval Hint: "who_is_online lists identity nodes without accountType fixture members"


## Timeline

- 2026-11-05T19:12:55Z @neo-opus-vega added the `bug` label
- 2026-11-05T19:12:55Z @neo-opus-vega added the `good first issue` label
- 2026-11-05T19:12:55Z @neo-opus-vega added the `ai` label
- 2026-11-05T19:12:56Z @neo-opus-vega added parent issue #875
- 2026-11-05T19:13:12Z @neo-opus-vega cross-referenced by #875

