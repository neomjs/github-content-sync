---
number: 20310
title: A canvas "neural" animation layer for the Portal's tickets TreeList
author: neo-opus-ada
category: Ideas
createdAt: '2026-11-05T11:55:23Z'
updatedAt: '2026-11-05T11:55:23Z'
closed: false
closedAt: null
routingDispositionSchemaVersion: discussion-routing-disposition.v1
routingDisposition: undetermined
routingDispositionReason: no-authoritative-lifecycle-marker
routingDispositionEvidence: []
contentTrust:
  projected: true
  quarantined: 0
  signals: []
conversationCompletenessSchemaVersion: discussion-conversation-completeness.v1
conversationComplete: true
conversationCommentCountObserved: 0
conversationCommentCountTotal: 0
conversationReplyCountObserved: 0
conversationReplyCountTotal: 0
---
> **Author's Note:** This was moved from issue #8541 by **Ada (Claude Opus 5.5)** during the 2026-11-05 backlog sweep (#15000), at the operator's request. Its history, including Euclid's 06-23 note that routed it out of code-ready status, stays on the issue.

Scope: low-blast

## The Concept
#8541 (Jan 2026): apply the Portal's "neural" canvas animation to the tickets TreeList in the left nav, which has thousands of items and infinite scroll, to show high-performance, off-main-thread animation at scale. Two ideas were named: a glowing indentation guide that traces the active branch, and a scanner effect that highlights the active ticket.

## The Rationale
It's still unbuilt. The tickets view uses the shared content side nav (`src/app/content/Container.mjs` and `src/app/content/TreeList.mjs`, with lazy child loading). The engine already renders canvas effects in a worker.

## Open Questions
- **OQ1:** Is this a showcase worth its upkeep, or decoration on a navigation surface that should stay calm?
- **OQ2:** If it's built, should it live in the shared content TreeList (every content section gets it) or only in the tickets view?
- **OQ3:** How does a canvas overlay stay aligned with a virtualized, infinitely scrolling list without per-frame cross-worker round trips?

## Graduation criteria
A yes on OQ1, a decided owner (OQ2), and a sketch for OQ3 → one ticket. A no → close this.

Ada (Claude Opus 5.5, Claude Code) · session 7fa48586-c5ef-4b2c-9d2a-84140ed7b756
