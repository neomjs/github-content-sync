---
id: 940
title: A working pull wake route reads undeliverable and unarmed
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-08T21:37:55Z'
updatedAt: '2026-10-08T22:19:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/940'
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
closedAt: '2026-10-08T22:19:12Z'
---
# A working pull wake route reads undeliverable and unarmed

## Context

F3 of the [#571 fix-first ledger](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6066806393). It is split out of #503, whose "inverse case" section moves here; #503 keeps its push-side ACs. On 2026-10-08 my seat's pull route `WAKE_SUB:06c1d367` (`SENT_TO_ME`, `harnessTarget: 'none'`, polled by the Stop-hook listener) delivered an idle wake in 12.6 s ([receipt](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6066238923)). At 21:31Z `manage_wake_subscription list` still described that same route, its `lastPollAt` 20:40Z, like this:

```
routeDeliverable: false
routeWithdrawalReason: "harnessTarget 'none' is not deliverable on the Shape-B path (only 'a2a-webhook' is); this seat cannot receive a container wake until it is migrated"
```

Since #562 (PR #752), the pull route is the Claude seats' target route rather than a legacy one, and #768 (PR #939) stops the Fleet from arming the push route beside it.

## The Problem

Two Memory Core surfaces still treat `a2a-webhook` as the only way a wake arrives:
1. `list` annotates every row with the receiver-manifest builder's skip decision. That builder skips a pull route by design, since nothing pushes it, so `list` calls a working route withdrawn, and its reason says the seat cannot receive wakes.
2. The healthcheck arming verdict mirrors the same builder gate. A seat whose only active route is its pull route reads `{armed: false, reason: 'unmigrated-target'}`.

A verifier who trusts either surface fails a healthy seat. That is the false-negative twin of #503's original false positive.

## The Architectural Reality

- `ai/daemons/wake/buildReceiverManifest.mjs` `wakeRouteWithdrawalReasonFor`: the builder's skip decision, `status` first, then `harnessTarget === 'a2a-webhook'`. It is correct for the manifest, since the receiver dispatches push routes only.
- `ai/services/memory-core/WakeSubscriptionService.mjs` `routeDeliveryAnnotationFor`: `list`'s per-row annotation, documented as a projection of that skip decision.
- `ai/services/memory-core/HealthService.mjs` `buildSubscriptionArmingBlock`: answers `unmigrated-target` when no active row is on `a2a-webhook`.
- `ai/daemons/wake/armSeatWakePull.mjs` `PULL_ROUTE`: a pull route is `SENT_TO_ME` on `harnessTarget: 'none'`. `pollDigest` serves it and stamps `lastPollAt`, which `list` already returns on the row.
- The `manage_wake_subscription` description in `ai/mcp/server/memory-core/openapi.yaml` states the same rule: "only `a2a-webhook` does".

## The Fix

1. `routeDeliveryAnnotationFor`: an active `harnessTarget: 'none'` row reads `{routeDeliverable: true, routeDelivery: 'pull'}`, and a row the builder publishes reads `{routeDeliverable: true, routeDelivery: 'push'}`. Every other row keeps the builder's reason. The row's `lastPollAt` is the pull route's evidence.
2. `buildSubscriptionArmingBlock`: a seat holding an active pull route reads `{armed: true, reason: 'pull'}`, whatever its push rows say, because the manifest build cannot affect a route it never carries.
3. `wakeRouteWithdrawalReasonFor`: the `none` reason says what is true, a pull route that its seat polls and the receiver never carries. Every other reason stays byte-stable.
4. The openapi description names the pull route.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `list` row `routeDeliverable` / `routeDelivery` (new) | `routeDeliveryAnnotationFor` | active pull → `true`/`'pull'`; published push → `true`/`'push'`; else `false` + the builder's reason | an absent `status` resolves through the status policy, as today | JSDoc + openapi | unit |
| healthcheck `features.wake.subscription` | `buildSubscriptionArmingBlock` | an active pull route → `{armed: true, reason: 'pull'}` | every other verdict unchanged | JSDoc | unit |
| the builder's skip reason for `none` | `wakeRouteWithdrawalReasonFor` | names the pull route | other reasons byte-stable | JSDoc | unit |

**Decision Record impact:** aligned-with ADR 0025. This is the detect side: the seat's state is reported truthfully, with no actuator.

## Acceptance Criteria

- [ ] AC-1: `list` reports an active `harnessTarget: 'none'` row `routeDeliverable: true, routeDelivery: 'pull'`, with no withdrawal reason; a degraded pull row keeps the status reason. Red-first.
- [ ] AC-2 (non-vacuity control): a published `a2a-webhook` row reads `routeDeliverable: true, routeDelivery: 'push'`, and `mcp-notifications` / `disabled` rows keep the builder's reason.
- [ ] AC-3: the healthcheck arming verdict for a seat whose active route is a pull route reads `{armed: true, reason: 'pull'}`. A seat with no active row keeps `no-active-subscription`, and a push seat keeps `deliverable` / `missing-signing-key`. Red-first.
- [ ] AC-4: the receiver manifest still never carries a pull route; only the skip reason's text changes.
- [ ] AC-5 (post-merge): on a plane running this revision, my `list` reads my pull route `routeDeliverable: true, routeDelivery: 'pull'`.

## Out of Scope

- Healthcheck `daemonRunning: false` / `no-pulse-file`. It measures the swarm-heartbeat lane (`heartbeat.alive`), which is off by default, not pull delivery; `#17647` owns that blindness.
- The healthcheck arming verdict being the cache filler's rather than the caller's: defect-note `MESSAGE:7a26d5e3`, next to #931.
- #503's push-side ACs, and the Fleet routes pane (#768 / PR #939).

## Related

#503 (split from) · #571 · #768 · #562 · #931

Live latest-open sweep: the latest 20 open Brain issues at 21:37Z; no equivalent.
A2A claim sweep: the last 30 messages, all read states; no claim on this surface (Grace's 18:50Z ledger note assigns F3 to Vega).
MC sweep: "a pull wake route reads routeDeliverable false and healthcheck unmigrated-target although it delivers wakes", 6 results. Eos's 2026-09-27 `mcp-notifications` trial met the same reason on a route that is not polled. Grace's `#17647` work confirms `daemonRunning` is the heartbeat lane. No prior decision against reading pull routes as deliverable.
Own-assignment sweep: 14 open; #503 holds this case today, and this ticket moves it out.
Structure map: `npm run ai:structure-map -- --files --loc` exit 0; no new file. The owners are `ai/services/memory-core/` and `ai/daemons/wake/`.

Origin Session ID: 7d3fc6b2-cee6-4f82-ba2c-103729d4047a
Retrieval Hint: "pull wake route harnessTarget none routeDeliverable false unmigrated-target routeDelivery pull poll-digest lastPollAt"


## Timeline

- 2026-10-08T21:37:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-08T21:37:57Z @neo-opus-vega added the `bug` label
- 2026-10-08T21:37:57Z @neo-opus-vega added the `ai` label
- 2026-10-08T21:38:03Z @neo-opus-vega added parent issue #503
- 2026-10-08T21:38:07Z @neo-opus-vega cross-referenced by #503
- 2026-10-08T21:42:35Z @neo-opus-vega cross-referenced by PR #941
- 2026-10-08T22:02:26Z @neo-gpt-emmy cross-referenced by PR #939
- 2026-10-08T22:19:12Z @tobiu referenced in commit `c30d921` - "feat(memory-core): a working pull wake route reads deliverable and armed by its seat's own poll (#940) (#941)

`list` annotated every row with the receiver-manifest builder's skip decision,
and that builder skips a pull route by design: nothing pushes it. So an active
`harnessTarget: 'none'` route, delivered by its seat's poll-digest, read
`routeDeliverable: false` with a reason saying the seat could not receive
wakes, and the healthcheck arming verdict, mirroring the same gate, read
`unmigrated-target` for a seat whose only route is its pull route.

`routeDeliveryAnnotationFor` now names the delivery: `push` for a row the
builder publishes, `pull` for an active `none` row, and the builder's reason
for every other row. The arming verdict reads `{armed: true, reason: 'pull'}`
for a seat holding an active pull route, whatever its push rows say. The
builder still never carries a pull route; its skip reason now says why. The
`manage_wake_subscription` description names both deliveries."
- 2026-10-08T22:19:13Z @tobiu closed this issue
- 2026-10-08T22:38:34Z @neo-opus-vega cross-referenced by #768
- 2026-10-08T23:07:53Z @neo-opus-vega cross-referenced by #611

