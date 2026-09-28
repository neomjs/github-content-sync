---
id: 616
title: A watermark-less pollDigest skips unread receipt-backed broadcasts
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T17:03:24Z'
updatedAt: '2026-09-28T18:12:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/616'
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
closedAt: '2026-09-28T18:12:37Z'
---
# A watermark-less pollDigest skips unread receipt-backed broadcasts

## Context

Brain #610 (for #561) makes a watermark-less `pollDigest` answer from current state instead of walking the log. `WakeSubscriptionService#_collectCurrentUnreadEvents` enumerates the owner's inbound edges through the edge store's target index, and it keeps only `SENT_TO`.

The shared evaluator `heartbeatPulseEvaluator.mjs#matchSentToMeEdge` surfaces two owner-targeted shapes. A direct message is a `SENT_TO` edge, unread while the MESSAGE carries no `readAt`. A receipt-backed fan-out broadcast is a per-recipient `DELIVERED_TO` edge, unread while the edge carries no `readAt`. Both edges target the owner, and the enumeration drops the second. This was found in Round 3 of #610's review.

## The Problem

A watermark-less poll reports the snapshot head as its watermark, and the next watermarked poll walks from there. So an unread, wake-eligible broadcast that was delivered before the first poll is never surfaced by pull delivery. That breaks `pollDigest`'s own contract: *"a missed poll re-includes anything still unread on the next call"*.

## The Architectural Reality

- `ai/services/memory-core/WakeSubscriptionService.mjs#_collectCurrentUnreadEvents` (added by #610): `if (edge?.type !== 'SENT_TO') continue;`
- `ai/services/memory-core/heartbeatPulseEvaluator.mjs#matchSentToMeEdge` has three branches: `DELIVERED_TO` to the owner, `SENT_TO` to the owner, and `SENT_TO` to `AGENT:*`, which defers to receipts when they exist.
- `isMessageWakeEligible` excludes only `wakeSuppressed` messages, so a waking broadcast is eligible.

## The Fix

Keep `DELIVERED_TO` beside `SENT_TO` in the enumeration. Both target the owner, and `match()` already decides what "unread" means for each shape, so the service gains no second copy of that rule.

## Acceptance Criteria

- [ ] AC-1: an unread, wake-eligible receipt-backed broadcast (a `DELIVERED_TO` edge to the owner, no `readAt`) surfaces on the first watermark-less call, and a read one does not. The arm is red on #610's head.
- [ ] AC-2: a wake-suppressed broadcast still does not surface.

## Out of Scope

- **Legacy `SENT_TO → AGENT:*` broadcasts without receipts.** They are pre-receipt residue. Reaching them would mean scanning the sentinel's whole inbound set, with an all-edges receipt check per message.
- **The delta walk**, which is unchanged.

## Related

#561 · #610 · #609

Live latest-open sweep: latest 20 open Brain issues at 2026-09-28 17:02:42Z, plus searches for watermark-less poll broadcast, pollDigest DELIVERED_TO and receipt-backed broadcast wake. No equivalent.
A2A in-flight sweep: the last 30 messages carry no claim on this.
MC sweep: no prior decision. @neo-kimi-iris's 2026-07-25 note confirms that receipt-backed broadcasts reach a recipient through its own `DELIVERED_TO` edge.
Own-assignment sweep: no overlap.
Structure map: `ai/services/memory-core` (`WakeSubscriptionService.mjs`).
Sequencing: after #610 merges, since this changes code that #610 introduces.
Origin Session ID: ec438462-8e98-425e-92c3-723cdc3c9704
Retrieval Hint: "watermark-less pollDigest DELIVERED_TO receipt-backed broadcast skipped"


## Timeline

- 2026-09-28T17:03:25Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T17:03:25Z @neo-opus-vega added the `bug` label
- 2026-09-28T17:03:25Z @neo-opus-vega added the `ai` label
- 2026-09-28T17:03:26Z @neo-opus-vega added the `agent-os` label
- 2026-09-28T17:04:31Z @neo-opus-vega cross-referenced by PR #610
- 2026-09-28T17:24:02Z @neo-opus-vega cross-referenced by PR #617
- 2026-09-28T18:12:37Z @tobiu referenced in commit `bf89f12` - "Merge pull request #617 from neomjs/vega/616-delivered-to-unread

fix(memory-core): a watermark-less poll surfaces unread receipt-backed broadcasts (#616)"
- 2026-09-28T18:12:38Z @tobiu closed this issue

