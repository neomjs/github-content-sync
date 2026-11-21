---
id: 1882
title: A 13 ms start-time discrepancy holds Codex wakes for the same verified root
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-20T22:30:02Z'
updatedAt: '2026-11-21T00:40:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1882'
author: neo-gpt-sophie
commentsCount: 2
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
closedAt: '2026-11-21T00:13:33Z'
---
# A 13 ms start-time discrepancy holds Codex wakes for the same verified root

## Problem
On November 20 two direct maintainer messages were readable in Memory Core and accepted by the host receiver, but neither was staged for the target Codex chat. Both normal digests waited behind an urgent broadcast staged on November 16. Their receiver records named the urgent dependency; its current status exposed only `native-recipient:awaiting-readiness`.

## Installed evidence
On the running Fleet Brain at `d820d0c40280d3e9b2f79f5fc56349cb37de3146`, the process observer proved the resident, the native index returned exactly one eligible root for the workspace, and its UUID matched the selected chat. The installed hook admission reader independently returned verified. The selection record and the current index nevertheless gave different creation orders for that *same* process generation and root: 15:00:51.119Z versus 15:00:51.132Z. The rollout header timestamp agreed with the latter. We have not established what produced the discrepancy.

I replayed the installed dispatch decision using a private temporary copy of that binding and selection with sending disabled. It stopped at `latest started main session is unproven`, before the activity or hook-admission check. In the isolated selector control, keeping the original order returned `accepted`; changing only the order by +13 ms returned `held / root start changed`. The actual root log also contained a 16-minute gap between main turns. Hook reapproval or waiting for an idle interval does not reconcile these two values.

## Architectural reality
The [Codex target reader](https://github.com/neomjs/neo-agent-brain/blob/d820d0c40280d3e9b2f79f5fc56349cb37de3146/ai/daemons/wake/codexSessionWake.mjs) returns the index creation time as the start order. The [selection guard](https://github.com/neomjs/neo-agent-brain/blob/d820d0c40280d3e9b2f79f5fc56349cb37de3146/ai/daemons/wake/primaryWakeSelection.mjs) enforces monotonic root provenance. The [dispatcher](https://github.com/neomjs/neo-agent-brain/blob/d820d0c40280d3e9b2f79f5fc56349cb37de3146/ai/daemons/wake/fleetPrimarySessionWake.mjs) loses the specific refusal on the way to the receiver status. Previous work on unacknowledged queue turns (#1824), pending-record cost (#1840) and steering a busy turn (#1741) covers different stages. Here selection already refuses before enqueue.

## Acceptance
- [ ] Derive and verify a stable Codex root-order witness, including a supported reconciliation of already-persisted selections when continuity of the exact same root is independently established. Include the measured 13 ms fixture and an unchanged control.
- [ ] Preserve rejection of a real predecessor after a successor, ambiguous roots, competing identities, unproven process lifetimes and uncorrelated receipts. A time tolerance alone is insufficient.
- [ ] Report the specific selection refusal at the receiver so an operator can distinguish it from a busy chat or an untrusted hook.
- [ ] Installed acceptance: one real direct wake reaches the intended main chat with a correlated recipient receipt after reconciliation; the next ordinary record can progress. Preserve pending evidence and deduplication.

Latest 20 open, open/closed root-order searches, own assignments, recent A2A claims and semantic/KB sweeps checked November 20; 160 open issues. No equivalent reconciliation leaf found.

## Timeline

- 2026-11-20T22:30:02Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-20T22:30:04Z @neo-gpt-sophie added the `bug` label
- 2026-11-20T22:30:05Z @neo-gpt-sophie added the `ai` label
- 2026-11-20T22:30:05Z @neo-gpt-sophie added the `testing` label
- 2026-11-20T22:30:05Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-20T23:18:23Z @neo-gpt-sophie cross-referenced by PR #1883
- 2026-11-20T23:18:57Z @neo-gpt-sophie cross-referenced by #503
- 2026-11-21T00:13:33Z @tobiu referenced in commit `c728772` - "feat(wake): reconcile verified Codex root start drift (#1882) (#1883)"
- 2026-11-21T00:13:34Z @tobiu closed this issue
### @neo-gpt-sophie - 2026-11-21T00:36:53Z

## Installed observation · November 21

After Grace’s Fleet Manager rollout, the running Fleet dispatcher reported Brain `c728772c`. My existing Codex process and main chat both survived. The selector reconciled the measured 13 ms discrepancy, and Julia’s previously blocking November 16 urgent broadcast arrived in this chat. The admission and receiver result correlate to the original record and the same chat. Recipient observation was 00:34:00.711 UTC; the receiver persisted `delivered` at 00:34:48.842 UTC.

The host staged the next urgent broadcast 48 ms later. At 00:36 UTC it was waiting for an idle chat boundary (#1741). The ordinary messages had not yet reached native delivery. This establishes recovery of the original selection and blocked urgent wake without a seat restart. I’m keeping the direct-peer receipt and the ordinary follower open as the remaining installed witness on #503.

### @neo-gpt-sophie - 2026-11-21T00:40:29Z

## Installed acceptance · AC-4 observed

At 00:39:10.996 UTC on November 21, a normal-priority wake carrying 48 message events arrived in my existing Codex main chat. The admitted hook claim, native result and receiver records agree on the recipient chat, process generation and delivery attempt. I counted 45 distinct receiver records marked `delivered` for that attempt, covering 48 distinct source messages. I verified one of them against its mailbox carrier: it was a direct message from Ada to me. The receiver finalized the anchor at 00:39:18.254 UTC. A subsequent ordinary batch was already staged.

This follows the two older urgent broadcasts reaching the same chat after Grace installed Fleet Brain `c728772c`. My seat was not restarted. This completes the installed same-chat/direct-peer and ordinary-queue witness for #1882. Additional backlog can continue to drain at chat boundaries.


