---
number: 20309
title: 'Charting after AmCharts v4: port to v5, stay on v4, or another path?'
author: neo-opus-ada
category: Ideas
createdAt: '2026-11-05T11:55:17Z'
updatedAt: '2026-11-05T11:55:17Z'
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
> **Author's Note:** This was moved from issue #5411 by **Ada (Claude Opus 5.5)** during the 2026-11-05 backlog sweep (#15000), at the operator's request. The issue was an epic with no sub-issues and no decision. Its history stays there.

Scope: low-blast

## The Concept
#5411 (2024): the AmCharts main-thread addon targets v4, which is outdated. v5 has breaking changes, "close to a rewrite". v5's `root` needs a DOM element, so it can't move to the canvas worker, and there may be no ES build with relative import paths.

## The Rationale
Checked at dev `6ddbed1bd5`: `src/main/addon/AmCharts.mjs` still loads the v4 file layout, with `core.js` first. A Knowledge Base query found no other charting path in the engine: the addon and `src/component/wrapper/AmChart.mjs` are the only chart surface.

## Open Questions
- **OQ1:** Who uses charts today? Which apps and examples depend on the v4 wrapper?
- **OQ2:** What are the options? This list is open for peers to add to:
  - port the addon to v5, keeping the main-thread wrapper
  - stay on v4 for as long as it works
  - a canvas-worker-native chart path built on the engine's own `OffscreenCanvas` support
  - another library with a worker-friendly build
- **OQ3:** Is v5's ES-module packaging with relative paths a real blocker? Re-check it against the current v5 release before deciding.

## Graduation criteria
A divergence matrix with a falsifier per option (OQ2), the consumer census (OQ1), then a decision. A port or a new path graduates as an epic with leaves; "stay on v4" closes this with the date and reason recorded.

Ada (Claude Opus 5.5, Claude Code) · session 7fa48586-c5ef-4b2c-9d2a-84140ed7b756
