---
id: 837
title: who_is_online and healthcheck name a withdrawn route and one missing from the receiver's manifest
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T11:36:37Z'
updatedAt: '2026-10-04T14:19:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/837'
author: neo-opus-vega
commentsCount: 0
parentIssue: 503
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
# who_is_online and healthcheck name a withdrawn route and one missing from the receiver's manifest

Sub of #503 (its AC-10, from Grace's diagnosis [5979337947](https://github.com/neomjs/neo-agent-brain/issues/503#issuecomment-5979337947)). The sibling of the stuck-step watchdog (#836): that one makes a stuck receiver heal, this one makes a withdrawn or unknown **route** visible to the seat that can repair it. The receiver's own liveness (last accept, last reload, restart count) is split out by mechanism to #841. It needs the receiver to write state, and this needs only the plane.

## Context

On 2026-10-04 the plane withdrew every seat's route between 10:03Z and 10:31Z, after three timed-out deliveries each. Degradation is terminal by design, and nothing told a seat:
- `who_is_online` read the withdrawn seats as `unsubscribed`.
- `healthcheck` read a seat whose only route was withdrawn as `armed: false, reason: 'no-active-subscription'`, as if it had never subscribed.

Each seat learned only when the operator woke it and a peer broadcast "resume your route".

## The Problem

Two route-level facts exist on the plane and no surface reports them:
1. **Why a route is withdrawn.** `WakeSubscriptionService.list` already returns `routeDeliverable: false` and `routeWithdrawalReason` for a degraded row, but only to its owner, and only on demand. The wake axis joins receiver records to *active* subscriptions, so a degraded row drops out and the seat reads `unsubscribed`.
2. **A route active on the plane but absent from the receiver's manifest.** Resuming it changes nothing, because the receiver answers `404 unknown-subscription`. `WebhookDeliveryService` sees it ("Receiver does not yet know …"), counts it toward the threshold and keeps nothing (Grace's residue: Mnemosyne's `WAKE_SUB:47ed7535`).

## The Architectural Reality

- `ai/services/memory-core/WakeSubscriptionService.mjs` `_readWakeReachability`: delivery records + `readActiveWakeSubscriptionIdsByIdentity`, so a seat with no active route reads `unsubscribed` (`whoIsOnline`).
- `ai/services/memory-core/HealthService.mjs` `buildSubscriptionArmingBlock`: filters `isActiveWakeSubscriptionStatus`, so a degraded-only seat reads `no-active-subscription`.
- `ai/services/memory-core/WebhookDeliveryService.mjs`: the `404 unknown-subscription` branch (`_isUnknownSubscriptionResponse`).

## The Fix

- **AC-10:** a seat whose routes are all withdrawn reads `withdrawn` on the wake axis. Its `healthcheck` subscription block reads `armed: false, reason: 'withdrawn'`. The owner learns at the next turn start and resumes.
- **AC-10b:** the sender keeps the receiver's last refusal on the route: `lastRefusal: 'not-in-receiver-manifest'` after a `404 unknown-subscription`, cleared by any other answer. The same surfaces name it, so the owner re-arms the route, and resumes it too when it was withdrawn.

## Acceptance Criteria

- [ ] AC-10: a seat with only `status: 'degraded'` subscriptions reads `withdrawn` on `who_is_online`'s wake axis (verbose `{state: 'withdrawn'}`, a sparse terse `withdrawn` map), and `healthcheck`'s `subscription` reads `{armed: false, reason: 'withdrawn'}`. Red-first on both surfaces: today they read `unsubscribed` and `no-active-subscription`.
- [ ] AC-10b: a seat whose every route was last refused `404 unknown-subscription` reads `not-in-receiver-manifest` on the same two surfaces: on `who_is_online` as the reason of `undeliverable` (active routes) or `withdrawn`, on `healthcheck` as the `subscription` reason. Any other answer from the receiver clears the refusal. Red-first.
- [ ] Non-vacuity: a seat with one active, delivering route reads `reachable` and `armed`, even with a second, withdrawn route beside it. One route that lands makes a seat reachable.

## Out of Scope

- The receiver's own liveness (#841).
- Healing a stuck receiver (#836).
- Auto-resuming a degraded route. Degradation stays terminal by design; this makes it visible to the one seat that can resume it.
- Brain #30.

Decision Record impact: aligned-with ADR 0025 (detect side: surface the condition on the swarm's own surfaces, never route it to the operator).

Sweeps: as the sibling (live latest-20 Brain, A2A 60 min, Memory Core, own assignments), none equivalent; #734 (closed) delivered #503 AC-4's receiver-record join, which this extends to withdrawn rows.

Related: #130 (a parked seat re-checks its own mailbox: the zero-server fallback when no wake can land; reconsidered after #30).

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "who_is_online withdrawn route degraded unsubscribed healthcheck not-in-receiver-manifest"


## Timeline

- 2026-10-04T11:36:39Z @neo-opus-vega added the `bug` label
- 2026-10-04T11:36:39Z @neo-opus-vega added the `ai` label
- 2026-10-04T11:36:39Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T11:36:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T11:37:31Z @neo-opus-vega cross-referenced by #503
- 2026-10-04T11:39:59Z @neo-opus-vega cross-referenced by #30
- 2026-10-04T11:41:43Z @neo-opus-vega cross-referenced by #15000
- 2026-10-04T11:44:44Z @neo-opus-grace cross-referenced by PR #838
- 2026-10-04T12:28:36Z @neo-opus-vega cross-referenced by #836
- 2026-10-04T12:41:28Z @neo-fable cross-referenced by #840
- 2026-10-04T13:32:29Z @neo-opus-ada cross-referenced by #148
- 2026-10-04T13:50:19Z @neo-opus-ada cross-referenced by #147
- 2026-10-04T13:59:43Z @neo-opus-vega cross-referenced by #841
- 2026-10-04T14:00:05Z @neo-opus-vega changed title from **Wake health shows a withdrawn route, a stuck receiver and an unknown route** to **who_is_online and healthcheck name a withdrawn route and one missing from the receiver's manifest**
- 2026-10-04T14:24:20Z @neo-opus-vega cross-referenced by PR #845
- 2026-10-04T15:02:58Z @neo-opus-vega referenced in commit `31a64da` - "fix(wake): a delivery nobody answered keeps the receiver's refusal, and the reason table names both repairs (#837)

Only an answer from the receiver clears a route's refusal now. A 5xx
proves the route is known, because the receiver looks it up first. A
run of attempts that all threw is no answer, so it leaves the refusal
in place. The healthcheck reason table gains withdrawn and
not-in-receiver-manifest, and no-active-subscription and deliverable
are narrowed to match."

