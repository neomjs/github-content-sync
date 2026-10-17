---
id: 1083
title: A delivered wake route reads reachable through any run of skips
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-17T20:07:05Z'
updatedAt: '2026-10-17T20:40:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1083'
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
closedAt: '2026-10-17T20:40:55Z'
---
# A delivered wake route reads reachable through any run of skips

## Context

The AC-7 audit on #503 (2026-10-17, [6240019035](https://github.com/neomjs/neo-agent-brain/issues/503#issuecomment-6240019035)) read each wake surface against the receiver's own records. #503's own subscription (`WAKE_SUB:54aaef3c`, the benched guest seat `@neo-preview`) read `reachable` with `consecutiveFailures: 0` on the healthcheck and in `who_is_online`. Its records show the last delivery on 09-29, then 29 consecutive `skipped` dispatches from 10-03 to 10-17 17:00Z. The same healthcheck showed Euclid's route `delivered` at 19:43Z with `lastAttemptedAt` on 10-16.

## The Problem

- Once a delivery earns a route `reachable`, nothing retires it. Skips are transparent to the failure streak in both directions, deliberately, so no record newer than the delivery can move the verdict. A receiver that keeps declining a route (stale presence, a target app that is not running) leaves it `reachable` indefinitely. That is #503's failure mode, reached through another outcome.
- `lastAttemptedAt` moves on skips and failures but not on deliveries, so a recovered route reports a last attempt older than its last delivery.

## The Architectural Reality

- `ai/services/memory-core/wakeDeliveryProjection.mjs`, `projectWakeDelivery`: pure, ordered newest first. A `skipped` record sets `lastAttemptedAt` and continues; the final state is `reachable` whenever a delivery was seen and no failure followed it.
- Consumers: `WakeSubscriptionService` (the healthcheck's `features.wake.delivery.subscriptions`) and `projectIdentityWakeReachability` (the `who_is_online` wake axis, where one reachable route makes a seat reachable). At `afae1ef3` nothing else in `ai/` reads `lastAttemptedAt` or `lastOutcomeReason`, and neither the Institution nor `neo` does.
- Design authority: the module's own summary says "this projection refuses to resolve in the healthy direction on absence of evidence … `reachable` is earned by an observed `delivered` record and nothing else"; #503 AC-7 says "Each says so, or stops saying it." A delivery followed only by skips is evidence about the past, resolved as healthy now.
- Kept from `df9c5792` (Eos): a skip never counts as a failure and never erases one.

## The Fix

- `STALE_SKIP_RUN = 3`: a run of that many skips, newer than every conclusion, makes a delivered route `unknown`, never `unreachable`. The verdict carries the run as an additive `consecutiveSkips` field.
- `lastAttemptedAt` also moves on a delivery.

| Surface | Authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `projectWakeDelivery` verdict | `wakeDeliveryProjection.mjs` | a delivery plus ≥ 3 newer skips → `unknown`; adds `consecutiveSkips`; `lastAttemptedAt` includes deliveries | a lone skip or a shorter run stays `reachable`; failures still decide | module JSDoc | unit arms |
| healthcheck `features.wake.delivery.subscriptions` | `WakeSubscriptionService` | passes the verdict through unchanged | — | — | AC-5 |
| `who_is_online` wake axis | `projectIdentityWakeReachability` | a route stale by skips no longer makes its seat reachable | any other reachable route still does | — | AC-5 |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1: A delivered route followed by `STALE_SKIP_RUN` or more newer skips reads `unknown`, with `consecutiveSkips` equal to the run. Red-first on `dev`.
- [ ] AC-2: Controls. A shorter run reads `reachable`, a later delivery closes the run (`consecutiveSkips: 0`), and a run after failures stays `unreachable` with the failures counted.
- [ ] AC-3: A delivery is an attempt, so `lastAttemptedAt` never predates `lastDeliveredAt`. Red-first on `dev`.
- [ ] AC-4: The existing projection and reader specs pass unchanged.
- [ ] AC-5 (post-merge): on the deployed plane, `WAKE_SUB:54aaef3c` reads `unknown` with its skip count, and a recovered route reports `lastAttemptedAt` ≥ `lastDeliveredAt`.

## Out of Scope

- #503 AC-7's other repairs: retired routes in the healthcheck map (no status filter or age bound), the name of the history field (`lastOutcomeReason` keeps the last failure reason by design), and pull routes in `who_is_online`.
- Recording a skip's reason on the receiver record.
- #503 AC-6 (an unattended turn start on the osascript arm).

## Avoided Traps

- **Counting skips as failures:** it manufactures an alarm on a healthy seat, and a skip run would read `undeliverable` with no failure to support it. Eos's argument stands.
- **A clock-based age bound:** the projection is pure by construction (no I/O, no clock). A count of newer skips needs neither.

## Related

Parent #503 (AC-7). The same surfaces: `who_is_online`'s wake axis (#734) and withdrawn routes (#837).

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-17T20:05Z (newest #1081); no equivalent. A2A in-flight claim sweep (the last 30 messages, all read-states): no claim on the wake projection besides my own `[lane-claim]` at 19:59Z. MC sweep: "wake subscription reads reachable while dispatches are skipped, stale route, last delivery weeks ago", 6 results (wake-poll turns), no prior decision found. Own-assignment sweep: 9 open; #503 is the parent, and #30 (the wake transport) does not overlap. Structure map: `ai/services/memory-core/wakeDeliveryProjection.mjs` is listed under `ai/services/memory-core`; no new module.

Origin Session ID: 964d6ca5-1709-49ac-a7a5-2e6ca64dd031

Retrieval Hint: "wake route reachable through skipped dispatches consecutiveSkips"


## Timeline

- 2026-10-17T20:07:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-17T20:07:06Z @neo-opus-vega added the `bug` label
- 2026-10-17T20:07:06Z @neo-opus-vega added the `ai` label
- 2026-10-17T20:07:07Z @neo-opus-vega added the `agent-os` label
- 2026-10-17T20:07:13Z @neo-opus-vega added parent issue #503
- 2026-10-17T20:08:37Z @neo-opus-vega cross-referenced by PR #1084
- 2026-10-17T20:40:55Z @tobiu referenced in commit `6f5155c` - "fix(wake): a delivery followed only by a run of skips stops reading reachable (#1083) (#1084)

The projection kept `skipped` transparent to the failure streak, as designed:
a skip is the receiver declining to dispatch, so it neither counts as a failure
nor erases one. But nothing ever retired the last delivery as evidence, so a
route the receiver kept declining read `reachable` indefinitely. On 2026-10-17
#503's own subscription (WAKE_SUB:54aaef3c) read reachable through 29
consecutive skips, 18 days after its last delivery on 09-29.

A run of STALE_SKIP_RUN (3) skips newer than every conclusion now makes a
delivered route `unknown` — never `unreachable`, since no failure was observed
— and the verdict carries the run as `consecutiveSkips`. A lone skip still
decides nothing, a later delivery closes the run, and a run after failures
leaves them counted.

A delivery is also an attempt now: `lastAttemptedAt` no longer predates
`lastDeliveredAt` on a recovered route (the healthcheck showed Euclid's route
delivered at 19:43Z with a last attempt on 10-16)."
- 2026-10-17T20:40:56Z @tobiu closed this issue

