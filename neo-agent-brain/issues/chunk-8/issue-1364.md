---
id: 1364
title: Broadcasts skip agents whose identity node is not in the in-memory graph cache
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T23:35:29Z'
updatedAt: '2026-10-29T23:39:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1364'
author: neo-opus-ada
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
# Broadcasts skip agents whose identity node is not in the in-memory graph cache

## Context

An `AGENT:*` broadcast reaches only the agents whose `AgentIdentity` node happens to be loaded in Memory Core's in-memory graph cache at send time. An agent whose node is in storage but not cached gets no delivery edge, and the broadcast never appears in its mailbox.

It surfaced chasing defect-note `139c4cc4`. There, `MailboxService.ListMessagesCost.spec` intermittently counts 500 of its 1000 seeded messages on CI: every broadcast missed the recipient, and CI's retry in a fresh worker passed.

## The Problem

`addMessage` stores a broadcast's send-time cohort (`routing.broadcastRecipients`, `MailboxService.mjs:879`) and links one delivery edge per member. The cohort comes from `getBroadcastAudience(sentBy)` (`MailboxService.mjs:2508`), which iterates `GraphService.db.nodes.items`, the cache, and never storage. The cache is lazy and evictable:
- it is cold after a restart;
- `Database#syncCache` removes nodes another process changed;
- tests clear it.

Measured 2026-10-29 on `dev@b6f24069`. Two `AgentIdentity` nodes are upserted. The recipient is then dropped from `db.nodes` the way `syncCache` drops a node, with `autoSave` off, so storage keeps the row; the arm asserts both facts. One broadcast is sent. On base the send writes no `DELIVERED_TO` edge to the recipient; with the fix it does. With the recipient cached, the same flow delivers 400/400.

Correction: a first probe removed the node with `autoSave` on. That writes the removal through and deletes the storage row, so its `totalCount 0` was no evidence of this bug.

## The Architectural Reality

- The canonical identity reader is `readAgentIdentityNodes(db)` in `ai/graph/agentIdentityParticipation.mjs`. It reads `Nodes` through the `idx_nodes_label` expression index, and wake delivery (`daemons/wake/daemon.mjs`), the heartbeat (`SwarmHeartbeatService`) and `checkAllAgentIdle` already take identities from it. The broadcast audience is the one reader that uses the cache.
- `getBroadcastAudience` also accepts a legacy `AGENT` label. Nothing in `ai/` or `test/` writes that label; this reader is its only reference.

## The Fix

`getBroadcastAudience` builds the cohort from `readAgentIdentityNodes(GraphService.db.storage.db)` and keeps its predicate:
- an `@` id;
- not the sender;
- `accountType` of `agent`, excluding `human` and `sentinel`.

The dead `AGENT` branch goes, which aligns the broadcast audience with the identity set wake delivery already uses. A red-first arm proves that a cold-cache recipient receives the broadcast.

## Acceptance Criteria

- [ ] AC-1: a broadcast reaches an agent whose `AgentIdentity` node is persisted but not in `db.nodes` (red on `dev`, green here).
- [ ] AC-2: the sender, `human` and `sentinel` identities stay out of the cohort.
- [ ] AC-3: the existing mailbox specs pass unchanged.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds.

## Out of Scope

- Participation (benched seats) in the broadcast audience; today's semantics are kept.
- The CI trigger that evicts the cost spec's recipient, which wasn't reproduced locally. This fix removes the cache dependence the flake needs.

## Related

defect-note `139c4cc4` (the CI flake). #88 (`AGENT:*` wake attention).

Live latest-open + closed sweep: latest open Brain issues + a search for broadcast audience / `getBroadcastAudience` / `AGENT:*` cohort at 2026-10-29T23:35:10Z. The hits (#88, #563) are adjacent, with no equivalent. A2A: no claim in the unread queue at 23:02Z. Structure map: N/A, since this changes one function and adds a spec arm.

Retrieval Hint: "broadcast audience in-memory graph cache cold recipient readAgentIdentityNodes getBroadcastAudience AGENT:* delivery"


## Timeline

- 2026-10-29T23:35:29Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T23:35:31Z @neo-opus-ada added the `bug` label
- 2026-10-29T23:35:32Z @neo-opus-ada added the `ai` label
- 2026-10-29T23:40:20Z @neo-opus-ada cross-referenced by PR #1365
- 2026-10-29T23:40:54Z @neo-opus-ada cross-referenced by #201

