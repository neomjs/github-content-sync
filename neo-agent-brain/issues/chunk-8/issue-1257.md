---
id: 1257
title: WakeSubscriptionService subscribe/bootstrap misses and orphans degraded routes
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-24T21:34:15Z'
updatedAt: '2026-10-25T03:21:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1257'
author: neo-gemini-julia
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
closedAt: '2026-10-25T03:21:57Z'
---
# WakeSubscriptionService subscribe/bootstrap misses and orphans degraded routes

## Context

During the 2026-10-24 host wake receiver freeze (`#1255`, where `PID 94810` deadlocked in `uv__threadpool_cleanup` from `09:03Z` to `20:52Z`), `WebhookDeliveryService._markDegraded` flipped active `a2a-webhook` seat subscriptions to `status: 'degraded'` after 3 consecutive timeout batches (`@neo-gemini-julia` `WAKE_SUB:6524cbcb` at `14:20:07Z`, `@neo-gemini-maryam` `WAKE_SUB:a39f9668` at `14:20:17Z`, plus `@neo-gpt-euclid`, `@neo-gpt-emmy`, and `@neo-gpt-sophie`).

When Fleet Manager booted seats at `19:19Z` (`WakeSubscriptionService.bootstrap` / `subscribe`), instead of reusing and resuming each seat's existing `WAKE_SUB` row on the identical canonical route key `(agentIdentity, trigger, filters, harnessTarget, {url})`, `WakeSubscriptionService` minted 5 brand-new `WAKE_SUB:<uuid>` nodes with new `signingKey` secrets (`2cf5f026`, `b2ed6dfe`, `95f1d8b1`, `d51263dc`, `c043a5d8`) and left all prior `status: 'degraded'` rows unretired in SQLite (`2–5` rows per seat).

- `Live latest-open sweep:` checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1255`, `#1254`, `#1253`, `#1252`, `#1251`, `#1250`, `#1248`, …) at `2026-10-24T21:32:07Z`; `#1255` (`PR #1256`) fixes the host daemon `receiver.mjs` `onStuck` `SIGKILL` exit and explicitly leaves the `mc-server` `WakeSubscriptionService` slice to this companion ticket.
- `A2A in-flight claim sweep:` checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-24T21:32:10Z`; `@neo-gemini-maryam` confirmed in `MESSAGE:a97ec67e-1da8-463a-9d44-c1183968b41a` (`21:15:11Z`) that `#1255` is scoped strictly to `ai/daemons/wake/receiver.mjs` and left this `mc-server` slice open to file.
- `MC sweep:` `query_raw_memories({ query: "WakeSubscriptionService degraded subscribe bootstrap duplicate WAKE_SUB reconcileDuplicateSubscriptions ROUTES_SQL withdrawn", nResults: 6 })` + `ask_knowledge_base({ query: "WakeSubscriptionService degraded subscribe bootstrap reconcileDuplicateSubscriptions duplicate WAKE_SUB", type: "ticket" })`, 6 + 5 results (`#19`, `#79`, `neo#11182`, `#50`, `neo#11993`); `neo#11182` added `_reconcileDuplicateSubscriptions` noting `"The exact root cause for the lookup-miss is unclear without runtime instrumentation"`.
- `Own-assignment sweep:` 1 open issue assigned to `@neo-gemini-julia` in `neomjs/neo-agent-brain` (`#1248`, `attributePreIdentityMemories.mjs` — unrelated).
- `Structure Map Gate:` ran `npm run ai:structure-map -- --files --loc`; owning files in `ai/services/memory-core/` (`WakeSubscriptionService.mjs`, `wakeSubscriptionStatusPolicy.mjs`, `readActiveWakeSubscriptionIdentities.mjs`).

## The Problem

`WakeSubscriptionService.mjs:554-560` (`bootstrap` comment from `neo#11182`) noted:
> *"The route-key idempotency check below is necessary but empirically not sufficient: across sessions, duplicates accumulate (`@neo-opus-ada` had 2 active subscriptions 2 days apart; `@neo-gpt` same pattern). The exact root cause for the lookup-miss is unclear without runtime instrumentation..."*

The 2026-10-24 incident exposed the exact root cause of both the route-lookup miss and the unretired duplicate accumulation:

1. **`_findActiveSubscriptionByRoute` (`:2439-2457`) and `_getCandidateSubscriptions` (`:2568-2587`) exclude `status: 'degraded'` rows:**
   - `_getCandidateSubscriptions` queries SQLite with `AND ${activeWakeSubscriptionStatusSql()}` (`COALESCE(status, 'active') = 'active'`), and `_findActiveSubscriptionByRoute` filters by `isActiveWakeSubscriptionStatus(subscription.status)`.
   - Whenever `WebhookDeliveryService._markDegraded` (`ai/services/memory-core/WebhookDeliveryService.mjs:282-288`) sets `status: 'degraded'` on a subscription after 3 failed delivery batches, that subscription becomes invisible to `_findActiveSubscriptionByRoute`.
   - On the very next `bootstrap()` or `subscribe()` call for that seat on the **exact same canonical route key**, `_findActiveSubscriptionByRoute` returns `null`, and `subscribe()` allocates a brand-new `WAKE_SUB:<uuid>` and mints a brand-new `signingKey` (`WakeSubscriptionService.mjs:1452-1462`).
   - This directly violates the design authority in `WakeSubscriptionService.mjs:1593-1600`:
     > *"The only escape was unsubscribe + subscribe, which allocates a **new subscription id** — and that id is what the manifest, the receiver route table, delivery receipts, and the degrade all index on. Recovery by re-identification is not repair. ... Minting there would make every defensive re-subscribe a silent key rotation, and a wrong rotation is worse than the bug it fixes: it invalidates a `routes.json` a live receiver already validated, turning a deaf seat into a deaf seat plus a stale manifest."*

2. **`_reconcileDuplicateSubscriptions` (`:2472-2521`) also excludes `status: 'degraded'` rows, leaving superseded `degraded` duplicates in `ROUTES_SQL` and `list()` forever:**
   - `_reconcileDuplicateSubscriptions(owner)` queries SQLite with `AND ${activeWakeSubscriptionStatusSql()}` (`:2480`).
   - Any subscription that degraded before a newer row was minted on the same canonical route key is never retired (`status: 'retired'`).
   - Meanwhile, `ROUTES_SQL` in `ai/services/memory-core/readActiveWakeSubscriptionIdentities.mjs:83-91` queries:
     ```sql
     AND ${resolvedWakeSubscriptionStatusSql()} IN ('active', 'degraded')
     ```
     and `manage_wake_subscription({ action: 'list' })` (`_listDurableSubscriptionsForOwner`, `:2657-2676`) returns all durable rows. As a result, superseded `degraded` rows remain permanently visible in `list()` and `ROUTES_SQL` until manually unsubscribed one by one.

## The Architectural Reality

- `Design authority:`
  1. `ai/services/memory-core/WakeSubscriptionService.mjs:1542-1546` (`resume` JSDoc): `"Degradation is deliberately terminal: a route whose endpoint refuses connections is marked status: 'degraded' and then never attempted again, which is what bounds the attempt count. Coming back is therefore an explicit operator act, never something the delivery path does on its own"`
  2. `ai/services/memory-core/WakeSubscriptionService.mjs:1593-1600` (`rotateKey` JSDoc): `"Recovery by re-identification is not repair. ... Minting there would make every defensive re-subscribe a silent key rotation ... it invalidates a routes.json a live receiver already validated"`
  3. `ai/services/memory-core/readActiveWakeSubscriptionIdentities.mjs:79-91` (`ROUTES_SQL`): `"A route is an active subscription or one the sender withdrew, which its owner can resume; a retired row is no route."`
- Calling `bootstrap()` or `subscribe()` at seat boot / recovery **is** an explicit caller/Fleet re-arm act (unlike background `WebhookDeliveryService` delivery). When a route with the same canonical route key (`_buildSubscriptionRouteKey`, `:2928-2938`) already exists in `status: 'active'` or `status: 'degraded'`, reusing and resuming that existing `WAKE_SUB:<uuid>` in place preserves its `subscriptionId` and `signingKey` (unless `a2a-webhook` lacks a server-issued `signingKey`) and clears `WebhookDeliveryService.clearDegraded(id)`.

## The Fix

1. **Shared SQL/JS predicate in `ai/services/memory-core/wakeSubscriptionStatusPolicy.mjs`:**
   - Export `WITHDRAWN_WAKE_SUBSCRIPTION_STATUS = 'degraded'`, `isReusableWakeSubscriptionStatus(status)` (`true` for `'active'` / absent or `'degraded'`), and `reusableWakeSubscriptionStatusSql(column = 'data')` (`${resolvedWakeSubscriptionStatusSql(column)} IN ('active', 'degraded')`), unifying the predicate already hand-written in `readActiveWakeSubscriptionIdentities.mjs:83-91` (`ROUTES_SQL`).
2. **Reconcile across both `active` and `degraded` rows in `_reconcileDuplicateSubscriptions(owner)` (`WakeSubscriptionService.mjs:2472-2521`):**
   - Query `AND ${reusableWakeSubscriptionStatusSql()}` ordered so `active` rows sort ahead of `degraded` rows within the same canonical route-key group (then by ` COALESCE(updatedAt, createdAt, '') DESC`).
   - Keep `[0]` and retire (`_retireSubscription(id)` -> `status: 'retired'`) all remaining `active` or `degraded` duplicates in that canonical route-key group.
   - Invoke `this._reconcileDuplicateSubscriptions(owner)` at the start of `subscribe()` (`:1421`) as well as `bootstrap()` (`:561`) so direct `subscribe` callers get the same duplicate-retirement guarantee.
3. **Reuse and resume existing `degraded` routes in place on `bootstrap()` / `subscribe()` (`WakeSubscriptionService.mjs:2439-2457, 2568-2587, 2776-2799`):**
   - Update `_getCandidateSubscriptions`, `_getCandidateSubscriptionsFromMemory`, and `_findActiveSubscriptionByRoute` to match `isReusableWakeSubscriptionStatus(subscription.status)` (`active` or `degraded`), preferring an `active` match if present and otherwise matching the newest `degraded` row with the same `_buildSubscriptionRouteKey`.
   - In `_refreshExistingSubscriptionRoute(subscription, {filters, harnessTargetMetadata})` (`:2776-2799`):
     - Preserve the existing `subscription.harnessTargetMetadata.signingKey` when `subscription.harnessTarget === 'a2a-webhook'` (only minting one if `signingKey` was absent), set `status: 'active'`, and call `WebhookDeliveryService.clearDegraded(id)` so an existing `degraded` route is cleanly resumed in place without allocating a new `subscriptionId` or rotating a valid `signingKey`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `isReusableWakeSubscriptionStatus` & `reusableWakeSubscriptionStatusSql` (`ai/services/memory-core/wakeSubscriptionStatusPolicy.mjs:69-100`) | `readActiveWakeSubscriptionIdentities.mjs:79-91` (`ROUTES_SQL`) | Admits both `active` (including absent) and `degraded` rows as live/resumable routes while excluding `retired` and unknown states | Unknown status tokens fail closed (`false`) | Module & function JSDoc | Unit spec in `WakeSubscriptionService.spec.mjs` |
| `_reconcileDuplicateSubscriptions(owner)` (`ai/services/memory-core/WakeSubscriptionService.mjs:2472-2521`) | `WakeSubscriptionService.mjs:2460-2466` + `ROUTES_SQL` | Groups owner's `active` and `degraded` rows by `_buildSubscriptionRouteKey`, keeps the newest (preferring `active` over `degraded`), and marks all older duplicates `status: 'retired'` | Returns `0` when `<= 1` row exists | Method JSDoc | Unit spec in `WakeSubscriptionService.spec.mjs` |
| `subscribe` / `bootstrap` / `_refreshExistingSubscriptionRoute` (`ai/services/memory-core/WakeSubscriptionService.mjs:548-617, 1421-1494, 2439-2457, 2776-2799`) | `WakeSubscriptionService.mjs:1542-1546, 1593-1600` | Reuses an existing `active` or `degraded` row matching the canonical route key, restores `status: 'active'`, calls `WebhookDeliveryService.clearDegraded(id)`, and preserves its `subscriptionId` and `signingKey` | Mints a new `WAKE_SUB:<uuid>` + `signingKey` only when no `active` or `degraded` row exists for the route key | Method JSDoc | Unit spec in `WakeSubscriptionService.spec.mjs` |

## Decision Record impact

`aligned-with` ADR 0002 (`learn/agentos/decisions/0002-phase3-wake-substrate-standards-alignment.md` §6.6); enforces the in-place recovery invariant documented in `WakeSubscriptionService.mjs:1593-1600`.

## Acceptance Criteria

- [ ] AC-1 `wakeSubscriptionStatusPolicy.mjs` exports `WITHDRAWN_WAKE_SUBSCRIPTION_STATUS` (`'degraded'`), `isReusableWakeSubscriptionStatus`, and `reusableWakeSubscriptionStatusSql`, and `readActiveWakeSubscriptionIdentities.mjs` (`ROUTES_SQL`) consumes them instead of a local duplicate.
- [ ] AC-2 Calling `bootstrap()` or `subscribe()` when a matching canonical route exists in `status: 'degraded'` reuses the existing `WAKE_SUB:<uuid>`, preserves its server-issued `signingKey`, sets `status: 'active'`, and calls `WebhookDeliveryService.clearDegraded(id)` instead of allocating a new `WAKE_SUB:<uuid>`.
- [ ] AC-3 `_reconcileDuplicateSubscriptions(owner)` runs on both `bootstrap()` and `subscribe()` and retires (`status: 'retired'`) superseded `degraded` as well as `active` duplicate rows within the same canonical route-key group (preferring `active` over `degraded`, then newest `updatedAt`/`createdAt`).
- [ ] AC-4 Unit tests in `test/playwright/unit/ai/services/memory-core/WakeSubscriptionService.spec.mjs` cover: (a) `subscribe()` / `bootstrap()` reusing and resuming a `degraded` route in place without rotating `signingKey`, and (b) `_reconcileDuplicateSubscriptions` retiring older `degraded` duplicates alongside `active` routes.

## Out of Scope

- Host wake receiver watchdog termination (`neomjs/neo-agent-brain#1255` / `PR #1256`).
- Silent background auto-resumption inside `WebhookDeliveryService.deliver()`: per `WakeSubscriptionService.mjs:1542-1546`, the background delivery loop must never auto-reactivate a `degraded` route on its own, as a crash-looping receiver would restore unbounded 4-attempt retry loops on every message.

## Avoided Traps / Gold Standards Rejected

- **Allocating a new `WAKE_SUB:<uuid>` on re-subscribe when the existing row is `degraded`:** As `WakeSubscriptionService.mjs:1593-1600` documents, allocating a new `subscriptionId` and `signingKey` on re-subscribe invalidates `routes.json` on a live receiver and orphans the `degraded` row in `ROUTES_SQL` (causing `who_is_online` and `list` to accumulate withdrawn zombie routes).
- **Retiring `degraded` rows across *different* route keys:** An agent may legitimately hold both a `SENT_TO_ME` route and a `TASK_STATE_CHANGED` route; reconciliation must remain strictly grouped by `_buildSubscriptionRouteKey(subscription)`.

## Related

- Companion to `neomjs/neo-agent-brain#1255` / `PR #1256` (`Wake receiver default onStuck deadlocks in uv__threadpool_cleanup`).
- Resolves the lookup-miss root cause noted in `neomjs/neo#11182` (`WakeSubscriptionService.mjs:554-560`).

unowned-rationale: Filed as the `mc-server` companion to `#1255` (`MESSAGE:a97ec67e`); filer's Sunday–Wednesday block is committed to Outcome 5 (Googlebot discovery enhancements: `middleware-v2#35`/`#36`, `neo#20088`/`#20089`), so any seat with a free Brain slot (or `@neo-gemini-maryam` after `#1256`) can self-select this one-PR leaf immediately.

Origin Session ID: 49d5ba98-613a-4b11-a63b-848f91397519
Retrieval Hint: "WakeSubscriptionService degraded subscribe bootstrap duplicate WAKE_SUB reconcileDuplicateSubscriptions ROUTES_SQL"


## Timeline

- 2026-10-24T21:34:16Z @neo-gemini-julia added the `bug` label
- 2026-10-24T21:34:17Z @neo-gemini-julia added the `ai` label
- 2026-10-24T21:34:17Z @neo-gemini-julia added the `model-experience` label
- 2026-10-24T21:34:17Z @neo-gemini-julia added the `agent-os` label
- 2026-10-24T21:41:19Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-24T21:56:23Z @neo-gemini-julia cross-referenced by PR #1259
- 2026-10-25T03:09:44Z @neo-gemini-julia referenced in commit `809f96f` - "docs(memory-core): document subscribe/bootstrap re-arm in resume() JSDoc (#1257)"
- 2026-10-25T03:21:57Z @tobiu referenced in commit `3410529` - "fix(memory-core): reuse and reconcile degraded wake subscriptions (#1257) (#1259)

* fix(memory-core): reuse and reconcile degraded wake subscriptions (#1257)

* docs(memory-core): document subscribe/bootstrap re-arm in resume() JSDoc (#1257)"
- 2026-10-25T03:21:57Z @tobiu closed this issue

