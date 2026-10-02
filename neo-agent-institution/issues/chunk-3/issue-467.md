---
id: 467
title: Give cockpit viewer-wake custody its own controller layer
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - refactoring
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-02T19:51:10Z'
updatedAt: '2026-10-02T20:30:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/467'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 24
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-02T20:30:34Z'
---
# Give cockpit viewer-wake custody its own controller layer

## Context

The #42 [headroom investigation](https://github.com/neomjs/neo-agent-institution/issues/42#issuecomment-5960139193) identifies a coherent viewer-wake custody seam inside the 990-line `cockpit/LivenessController.mjs`. This follows #24's responsibility-based decomposition and the existing ReadingSurfaces/Liveness controller layers.

## The Problem

LivenessController owns both bounded roster/activity/system reads and the viewer's wake-stream custody/projection. The wake cluster's two fields are confined to that responsibility; retaining it here leaves only ten lines below the app-file error boundary. The purpose is to give this distinct lifecycle an explicit home, not shorten unrelated methods or change wake behavior.

## The Architectural Reality

The final cockpit Controller supplies a fresh `bridge` getter. `ensureViewerWakeStream` keeps one consumer per bridge identity; `onViewerWakeSignal` writes the provider-owned feed; `stampViewerWake` copies the consumer's observations into provider data. The existing `fleetWakeStreamConsumer` module owns transport parsing/reconnect, not cockpit provider writes.

Container, liveness ticks and reconnect invoke the inherited seam. Custody-heal callbacks re-drive reconnect. `stopLiveness` is a separate timer/visibility stop-restart operation; full destruction also stops the wake consumer.

## The Fix

Add `view/fleet/cockpit/ViewerWakeController.mjs` extending `Neo.controller.Component`, and make LivenessController extend it. Move `getViewerWakeFeed`, `ensureViewerWakeStream`, `onViewerWakeSignal`, `stampViewerWake`, the two consumer/bridge custody fields and their teardown into that base layer.

Keep one final controller instance and all inherited method names. LivenessController keeps `onLivenessTick`, the generic `LivenessCadence.READS` iteration, reconnect orchestration, custody-heal generation fencing and timer-only stop semantics. Its destroy stops liveness before chaining through the wake owner's destroy.

Prescription checked: this is a cockpit controller concern, not transport logic. Structural fast-path: another controller responsibility layer beside the existing ReadingSurfaces/Liveness pair; no new directory role. The tradeoff is one additional inheritance level, explicitly retaining the final controller's bridge contract.

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Consumer custody | current ensureViewerWakeStream | one consumer per current bridge; same pollDigest capability | replacement/lost capability stops old consumer | class/method summaries | existing reuse/swap/loss controls |
| Feed and provider projection | current signal/stamp methods | retain bounded feed and verbatim liveness/catch-up observations | no envelope fabricates no row; destroyed owner is inert | moved summaries | viewerWake unit + NL witness |
| Caller integration | Container/tick/reconnect/custody-heal | same inherited calls on one final instance | generic READS iteration stays extensible | layer summaries | cadence/custody regression controls |
| Destruction | controller lifecycle | stop consumer on final teardown | timer-only stop remains independent; no duplicate consumer shutdown | destroy summaries | explicit real-controller teardown control |

## Acceptance Criteria

- [ ] The whole custody/projection cluster leaves LivenessController without forwarding copies, leaving measurable headroom; no consumer instance or provider Store is duplicated.
- [ ] Existing capability absence, reuse, bridge replacement, signal, observation and poll-digest behavior remains unchanged.
- [ ] The generic read iteration, cadence, reconnect and custody-heal generation fence remain in place and their regressions pass.
- [ ] A real final-controller lifecycle control proves destruction stops the owned consumer; timer-only stop/restart keeps its existing meaning.
- [ ] The served viewer-wake witness and relevant visual inputs agree on the final dependency base; input stamp refreshed without accepting an unexplained image change.

## Out of Scope

Brain wake routing/subscriptions, transport retry/parser behavior, credentials, open-work projection, new cadence keys, selection ownership and installed Fleet changes.

## Avoided Traps

No second controller object, wrapper facade, generic service abstraction or provider. No new wake authority in Body; the consumer's observations remain authoritative. No READS-loop relocation.

Decision Record impact: none — applies the existing controller decomposition law.

Related: #24 (parent), #42 (measurement), #449 (independent consumer).
Ada confirmed via `d6637b5b-ab71-4b61-9edc-8929f6e50982` that her open-work lane touches the cadence util and final Controller, not this file/cluster. Its one condition—retain the generic READS iteration—is part of this scope.

Sweeps: live latest 20 open issues and recent all-state A2A checked immediately before filing; no equivalent. Exact all-state viewer-wake/controller search found the measurement parent but no repair. MC queries returned unrelated historical wake records, not an extraction ruling. Own assigned #42 already carries the measurement; this leaf carries its code. Brain structure-map runner: N/A in this consumer; existing cockpit controller siblings establish placement.

Origin Session ID: 8d1cf4b5-75d2-4880-8358-873e0ac47fe0
Retrieval Hint: LivenessController viewerWakeConsumer viewerWakeBridge viewer-wake custody headroom


## Timeline

- 2026-10-02T19:51:10Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-02T19:51:11Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-02T19:51:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-02T19:51:11Z @neo-gpt-emmy added the `ai` label
- 2026-10-02T19:51:12Z @neo-gpt-emmy added the `refactoring` label
- 2026-10-02T19:51:42Z @neo-gpt-emmy added parent issue #24
- 2026-10-02T20:10:11Z @neo-gpt-emmy cross-referenced by PR #468
- 2026-10-02T20:18:00Z @neo-gpt-emmy cross-referenced by #42
- 2026-10-02T20:30:34Z @tobiu referenced in commit `85a8a8a` - "chore(agentos): isolate cockpit viewer-wake custody (#467) (#468)"
- 2026-10-02T20:30:35Z @tobiu closed this issue

