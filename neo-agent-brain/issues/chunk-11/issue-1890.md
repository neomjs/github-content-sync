---
id: 1890
title: Skip pending wake/digest records whose sourceEventIds are handled
state: CLOSED
labels:
  - bug
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-21T12:09:59Z'
updatedAt: '2026-11-22T11:01:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1890'
author: neo-gemini-julia
commentsCount: 1
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
closedAt: '2026-11-22T11:01:58Z'
---
# Skip pending wake/digest records whose sourceEventIds are handled

Not offered as a `good first issue` / `help wanted`: reproducing and verifying end-to-end spans the HMAC-signed Memory Core ↔ graphless host wake receiver boundary (`POST /wake/settle`, `WakeReceiverState`, and `primary-session` `inflight` lock safety).

## Context

Observed across three consecutive turns on `@neo-gemini-julia` (`2026-11-21T11:39Z`–`11:55Z`, session `664ca229-4bbd-4e00-ae41-008707688181`) after autonomous push wakes resumed on `runtime/c3ca946b` (`#1743` / PR `#1866`):

- Turn 31 (`11:39Z`), Turn 32 (`11:48Z`), and Turn 33 (`11:55Z`) each fired a native `[NEO_WAKE_V1:<nonce>] [WAKE]` prompt replaying **5-day-old `2026-11-16`** `wake/digest` envelopes (`neomjs/neo#20910`, `neomjs/neo#21052`, and `neo-stage#817`/`#818` + `benchmarks#9`–`#12`) at 60-second `DRAIN_RETRY_INTERVAL_MS` intervals while `list_messages({status: 'unread', projection: 'counts'})` returned **`unreadCount: 0`** on every turn.
- Captured in `defect-note` `MESSAGE:c7ab081f-d0be-43e5-9625-f62e8199841c` (`fingerprint: 87b62271baf815c7`) and promoted here after the third consecutive reproduction.

## The Problem

`CoalescingEngineService#_reconcileMessageWakeReadState` (`ai/services/memory-core/CoalescingEngineService.mjs:839-880`) suppresses already-read `wake/sent_to_me` events **before** flushing a `wake/digest` over Shape B (`WebhookDeliveryService#deliver`). However, once a `wake/digest` has been flushed and durably accepted as `state: 'pending'` in host `WakeReceiverState` (`ai/daemons/wake/receiverState.mjs:95-120`) — for example, the `189` `pending` records queued while `@neo-gemini-julia` waited on `native-recipient:awaiting-readiness` (`#1840`), or digests queued while a seat is `context-gate` deferred — subsequent mailbox consumption (`mark_read` / terminal `transition_task` during an active turn or `/context-recovery`) never reaches the host receiver.

Even after ordinary primary-session catch-up batching (`#1863` / PR `#1878` for Codex; `#1887` / PR `#1888` for Antigravity) merges and deploys:

1. A returning seat whose inbox is already at `unreadCount === 0` still receives `ceil(N / 48)` multi-kilobyte catch-up wake turns (`4` turns × up to `16 KiB` for `189` historical records) containing zero actionable unread items, burning `512K`-context KV cache and interrupting active lanes.
2. Urgent `pending` records (`!isUrgentWakeRecord(record)` is false, so `fleetPrimarySessionWake.mjs:98` excludes them from `eligibleBatchRecord`) and non-batch adapters (`osascript`, `tmux`, `claude-courier`, `opencode-server`, `kimi-server`) still fire **one LLM turn per already-handled record** onto an empty inbox.

## The Architectural Reality

- **Graphless host receiver boundary (`ai/daemons/wake/receiver.mjs:5-10`):** `receiver.mjs` verifies HMAC-SHA256 over exact request bytes (`verifyWakeSignature`, `X-Neo-Wake-*` headers) and owns `WakeReceiverState` (`records/<recordKey>.json`), importing **no** graph, SQLite, Memory Core config, or database path.
- **Canonical source-event provenance (`ai/services/memory-core/CoalescingEngineService.mjs:1000-1049`):** Every emitted `wake/digest` persists `sourceEventIds` on both the envelope (`envelope.payload.sourceEventIds`) and the accepted receiver record (`record.sourceEventIds`, `receiver.mjs:984-997`), populated via `_getSourceEventId` (`MESSAGE:<uuid>`, `TASK:<uuid>`, `HEARTBEAT_PULSE...`, `PERMISSION...`).
- **Route-admission trap on `POST /wake` (`ai/daemons/wake/receiver.mjs:972` & `ai/services/fleet/fleetWakeReceiver.mjs:125`):** `POST /wake` enforces `envelope.eventType === routeAdmits(route)` (`'wake/digest'` or `'session/rotate'`) and returns `409 signed-route-mismatch` on any other `eventType`, which `WebhookDeliveryService#deliver` (`ai/services/memory-core/WebhookDeliveryService.mjs:187`) treats as a client error that marks the subscription **`degraded`**. Settled-source notifications MUST therefore use a dedicated `POST /wake/settle` path where `404` on un-upgraded receivers or `fleetWakeReceiver` is a non-degrading capability no-op.
- **Native in-flight redemption guard (`ai/daemons/wake/fleetPrimarySessionWake.mjs:165-186`):** An already-injected native wake (`inflight.status === 'unknown'` or `urgentInflight.status === 'unknown'`) requires its anchor `recordKey` and any frozen `activeAttempt.members` to remain `pending` or `dispatching` until `consumeNativeWakePrompt` or receipt reconciliation finishes; settling an active unredeemed `inflight` member mid-flight would deny hook redemption.
- **Reachability projection invariant (`ai/services/memory-core/wakeDeliveryProjection.mjs:48, 153-155, 188`):** `projectWakeDelivery` flips a `reachable` subscription to `unknown` once `consecutiveSkips >= STALE_SKIP_RUN` (`3`) newer than `lastDeliveredAt` (designed for `context-gate:active` skips of untested routes). Transitioning `3+` historical `pending` records to `skipped` (`outcomeReason: 'source-events-already-handled'`) MUST NOT increment `consecutiveSkips` and falsely downgrade a live seat to `unknown`.
- Design authority: "`CoalescingEngineService` — Suppressed N already-read message wake event(s) for <identity> at flush" (`ai/services/memory-core/CoalescingEngineService.mjs:873-874`) together with `ai/daemons/wake/receiver.mjs:8-9` ("It imports no graph, SQLite, Memory Core config, or database path. Container Memory Core owns matching/coalescing/retry; this process owns only the local final mile") and `#1840` Avoided Traps ("Expiring old pending wakes to shorten the pass. It hides the cost and drops wakes a returning seat should get").

## The Fix

1. **Signed `POST /wake/settle` on `ai/daemons/wake/receiver.mjs`:**
   - Accept HMAC-signed `POST /wake/settle` requests authenticated with the route's existing server-issued `signingKey` (`verifyWakeSignature` over exact raw bytes; `envelope.eventType === 'wake/settle'`, `schemaVersion === '1.0'`, matching `subscriptionId` + `agentIdentity`, rejecting `session/rotate` routes).
   - Under receiver drain/state serialization, scan `state === 'pending'` `wake/digest` records for `subscriptionId`: when `record.sourceEventIds` is non-empty, every ID is a settleable message/task source ID (`MESSAGE:*` / `TASK:*`, never `HEARTBEAT_PULSE*`, `PERMISSION*`, or `'unknown'`), **every** ID is in the signed `settledSourceEventIds` set, and the record is **not** bound in an active `primary-session` `inflight` / `urgentInflight` attempt (`recordKey` or `members`), transition `pending -> skipped` with `outcomeReason: 'source-events-already-handled'` and `dispatchFinishedAt`.
2. **Memory Core settle emission (`WebhookDeliveryService.mjs`, `MailboxService.mjs`, `WakeSubscriptionService.mjs`):**
   - Add `WebhookDeliveryService#deliverSettle(subscription, envelope)` posting to `<webhookOrigin>/wake/settle` with the existing `X-Neo-Wake-*` headers; treat `404` (`not-found` / `unknown-subscription`) and network timeouts as non-degrading best-effort outcomes (never calling `_markDegraded`).
   - Emit settled `MESSAGE:*` / terminal `TASK:*` IDs for the recipient's active `a2a-webhook` wake subscriptions when `MailboxService#markRead` stamps `readAt` (`MailboxService.mjs:5512`) or `MailboxService#transitionTask` reaches a terminal state (`Completed`, `Canceled`, `Failed`, `Rejected`, `Expired`; `MailboxService.mjs:6661`) when the backing message is also read (or task-only).
   - **Historical backlog sweep:** When `readWakeDelivery` (`ai/services/memory-core/wakeDeliveryReader.mjs:88`) is readable and a seat's `MailboxService#listMessages` (`status: 'unread'`, `projection: 'counts'`) observes `unreadCount === 0` (or after `markRead`), reconcile any `pending` `wake/digest` records for that seat whose `sourceEventIds` are already durably read/terminal in Memory Core via one bounded `deliverSettle` call, clearing pre-existing multi-day queues on first contact.
3. **Delivery projection preservation (`ai/services/memory-core/wakeDeliveryProjection.mjs`):**
   - Keep `skipped` records with `outcomeReason === 'source-events-already-handled'` transparent to both `consecutiveFailures` and `consecutiveSkips` in `projectWakeDelivery` (`wakeDeliveryProjection.mjs:142-158`) so settling a multi-record backlog never trips `STALE_SKIP_RUN` (`3`) on a reachable seat, while `projectMessageDelivery` (`MESSAGE_RANK`) continues to report `skipped` for messages superseded before delivery and `delivered` if any route delivered them.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `POST /wake/settle` (`ai/daemons/wake/receiver.mjs`, new endpoint) | ADR 0002 §6.2.3 + `#1840` + this ticket | Verifies `X-Neo-Wake-*` HMAC over raw bytes against `route.signingKey`; transitions eligible `pending` `wake/digest` records whose non-empty `sourceEventIds` are all in `settledSourceEventIds` (and not active in `primary-session` `inflight`/`urgentInflight`) to `skipped` (`outcomeReason: 'source-events-already-handled'`) and returns `200 {status: 'settled', skippedCount}` | Invalid signature → `401`; unknown sub → `404`; route/schema mismatch → `409`; records with any unsettled, `HEARTBEAT_PULSE*`, `PERMISSION*`, or active `inflight` ID stay `pending` | JSDoc in `receiver.mjs` | Unit spec driving real HTTP `createWakeReceiver` + `primary-session` `inflight` guard |
| `WebhookDeliveryService#deliverSettle` (`ai/services/memory-core/WebhookDeliveryService.mjs`, new method) | `WebhookDeliveryService#deliver` (`WebhookDeliveryService.mjs:91`) + `fleetWakeReceiver.mjs:125` | Derives `/wake/settle` from `harnessTargetMetadata.url`, signs exact JSON bytes with `signingKey`, and dispatches bounded settle payload without mutating subscription degradation counters | `404` (older `receiver.mjs` or `fleetWakeReceiver`), `409`, or network error logs debug/warn and returns `'skipped'`/`'failed'` **without** calling `_markDegraded` | `@summary` JSDoc on `deliverSettle` | Unit spec asserting `404` and `409` on `/wake/settle` leave subscription `active` (`consecutiveFailures === 0`) |
| `MailboxService#markRead`, `MailboxService#transitionTask`, `MailboxService#listMessages` (`ai/services/memory-core/MailboxService.mjs:5512,6661,4623`) | `CoalescingEngineService#_reconcileMessageWakeReadState` (`CoalescingEngineService.mjs:839`) | Publishes newly settled `MESSAGE:*` / terminal `TASK:*` source IDs to the recipient's active `a2a-webhook` wake routes, and reconciles readable historical `pending` receiver records when `unreadCount === 0` | Missing/unconfigured `wakeReceiverRecordsDir` skips historical directory scan while incremental `markRead`/`transitionTask` settle still runs; settle errors never fail the MCP call | JSDoc in touched Memory Core service methods | Unit spec covering incremental `markRead` settle + historical `189`-record `unreadCount === 0` sweep |
| `projectWakeDelivery` (`ai/services/memory-core/wakeDeliveryProjection.mjs:101`) | `wakeDeliveryProjection.mjs:38-48` (`STALE_SKIP_RUN`) | Treats `state === 'skipped'` with `outcomeReason === 'source-events-already-handled'` as supersession rather than untested silence (`consecutiveSkips` unchanged; `reachable` preserved when `sawDelivered`) | Other `skipped` reasons (e.g. `context-gate:active`) retain `STALE_SKIP_RUN = 3` retirement behavior | JSDoc update in `wakeDeliveryProjection.mjs` | Unit spec in `wakeDeliveryProjection.spec.mjs` with `1` `delivered` + `10` newer `source-events-already-handled` skips |

Decision Record impact: `aligned-with neomjs/neo-agent-brain#adr:0002-phase3-wake-substrate-standards-alignment`

## Acceptance Criteria

- [ ] `ai/daemons/wake/receiver.mjs` serves `POST /wake/settle` behind exact-byte `verifyWakeSignature` validation and transitions `pending` `wake/digest` records for that `subscriptionId` to `skipped` (`outcomeReason: 'source-events-already-handled'`) if and only if `record.sourceEventIds` is non-empty, contains only settleable message/task IDs, has every element present in `settledSourceEventIds`, and is not part of an active `primary-session` `inflight` or `urgentInflight` grant (`recordKey` or frozen `members`), while keeping zero SQLite/Memory Core imports in `ai/daemons/wake/receiver.mjs`.
- [ ] Partial-digest safety: a `pending` `wake/digest` record carrying `['MESSAGE:a', 'MESSAGE:b']` stays `pending` when only `'MESSAGE:a'` is settled, and transitions to `skipped` once `'MESSAGE:b'` is also settled; records containing `HEARTBEAT_PULSE*` or `PERMISSION*` source IDs are never settled via `POST /wake/settle`.
- [ ] `WebhookDeliveryService#deliverSettle` and the Memory Core mailbox/task hooks emit signed `wake/settle` envelopes on `markRead` / terminal `transitionTask` and reconcile readable historical `pending` records when a seat's unread count is `0`, without degrading subscriptions when an endpoint answers `404` or `409` (including `fleetWakeReceiver.mjs`).
- [ ] `projectWakeDelivery` (`ai/services/memory-core/wakeDeliveryProjection.mjs`) preserves `state: 'reachable'` when `3` or more `skipped` records with `outcomeReason: 'source-events-already-handled'` conclude after a `delivered` record, while preserving `STALE_SKIP_RUN = 3` for `context-gate` skips.
- [ ] Playwright unit specs (`npm run test-unit`) cover HMAC admission/rejection on `POST /wake/settle`, partial vs complete `sourceEventIds` settlement, active `primary-session` `inflight` member protection, `404`/`409` non-degradation in `WebhookDeliveryService`, and `projectWakeDelivery` reachability preservation.

## Out of Scope

- Ordinary catch-up batching for unread `primary-session` wakes (`#1863` / PR `#1878`, `#1887` / PR `#1888`).
- Blocked-drain CPU/log amplification (`#1840` / PR `#1881`), urgent active-turn steering (`#1741`), and autonomous wake-plane self-healing (`#1373`).
- Expiring unread `pending` wakes by wall-clock TTL.

## Avoided Traps

- **Wall-clock TTL / age-based expiration of `pending` wakes (`#1840` Avoided Trap):** Drops genuinely unread wakes that a returning offline or context-gated seat must still receive. Only positive Memory Core attestation that every `sourceEventId` is handled permits skipping.
- **Importing SQLite or `GraphService` into `ai/daemons/wake/receiver.mjs`:** Violates the graphless host-daemon boundary (`receiver.mjs:5-10`) and breaks host-edge startup isolation.
- **Sending `eventType: 'wake/settle'` to `POST /wake`:** Triggers `409 signed-route-mismatch` on `fleetWakeReceiver.mjs:125` and older `receiver.mjs:972`, causing `WebhookDeliveryService#deliver` (`line 187`) to mark healthy subscriptions `degraded`.
- **Incrementing `consecutiveSkips` on `source-events-already-handled`:** Would trip `STALE_SKIP_RUN = 3` (`wakeDeliveryProjection.mjs:48, 188`) and falsely report a live, active seat as `unknown` after clearing a backlog.

## Related

- Complements `#1887` (PR `#1888`), `#1863` (PR `#1878`), `#1840` (PR `#1881`), `#1824` (PR `#1825`), `#1743` (PR `#1866`), and `#1373`.
- Protects `Arm A1` `512K`-context token budget tracked on `#1853`.

unowned-rationale: Maintainer-scoped Agent OS wake/security boundary lane; ready for pickup once `#1888` merges (`@neo-gemini-julia` currently driving Engine `v14` cut queue).

Live latest-open + closed sweep: checked latest 20 open + open and closed search (`wake receiver pending sourceEventIds`, `already read`, `wake digest`; Out of Scope / Avoided Traps read on `#1887`, `#1863`, `#1840`, `#1824`, `#1373`) at `2026-11-21T12:07Z`; no open or closed duplicate/exclusion found (`#1840` and `#1887` exclude wall-clock expiry and direct SQLite coupling in `receiver.mjs`, which this ticket preserves via signed `POST /wake/settle`). Backlog band: `162` open (red); `#1887` covers unread batching only (`Resolves #1887` in PR `#1888`), so settled-source supersession requires its own 1-PR leaf.
MC sweep: `"multi-day old wake digest replay unreadCount 0 already read mailbox empty"` (5 results) + prior session sweeps (`90dcc4eb`, `6d185a99`, `167e0a91`), no prior exclusion found.
Own-assignment sweep: `2` open (`#1887` Antigravity idle-boundary batching with PR `#1888` approved; `#1230` Claude Desktop `ccd-session-presets.json`), bodies read, none owning post-accept settled-source skip.
A2A sweep: checked latest 30 messages across all read states (`2026-11-21T11:30Z`–`12:07Z`); promotes own `defect-note` `MESSAGE:c7ab081f-d0be-43e5-9625-f62e8199841c`, no competing `[lane-claim]` or `[lane-intent]`.
Structure map: `npm run ai:structure-map -- --files --loc` run in `neomjs/neo-agent-brain`; `ai/daemons/wake` (`receiver.mjs`, `fleetPrimarySessionWake.mjs`) and `ai/services/memory-core` (`WebhookDeliveryService.mjs`, `MailboxService.mjs`, `wakeDeliveryProjection.mjs`) own the two halves; no new `.mjs` file required.

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181
Retrieval Hint: "multi-day old wake/digest replay unreadCount 0 sourceEventIds already read POST /wake/settle source-events-already-handled STALE_SKIP_RUN"

## Timeline

- 2026-11-21T12:10:01Z @neo-gemini-julia added the `bug` label
- 2026-11-21T12:10:01Z @neo-gemini-julia added the `ai` label
- 2026-11-21T12:10:01Z @neo-gemini-julia added the `performance` label
- 2026-11-21T12:10:01Z @neo-gemini-julia added the `model-experience` label
- 2026-11-21T12:10:02Z @neo-gemini-julia added the `agent-os` label
- 2026-11-21T13:45:56Z @neo-gemini-julia cross-referenced by #1853
- 2026-11-22T03:33:54Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-11-22T03:34:29Z

## Ticket Intake (`valid-as-written` — Accept & Sharpen)

Verified on `origin/dev` (`d6f43108`, with `#1878`, `#1881`, and `#1888` merged):

- **Prescription checked:** `ai/daemons/wake/receiver.mjs`, `ai/daemons/wake/primaryWakeSpool.mjs`, `ai/services/memory-core/{WebhookDeliveryService,MailboxService,WakeSubscriptionService,wakeDeliveryReader,wakeDeliveryProjection}.mjs` — own the concern (`ADR successor-risk: adr-aligned` against `adr:0002-phase3-wake-substrate-standards-alignment` and `adr:0019-aiconfig-reactive-provider-ssot`).
- **Two `d6f43108` wire/spool sharpenings folded into implementation:**
  1. **Envelope breakdown check alongside prefix whitelist (`CoalescingEngineService.mjs:912-1049`, `heartbeatPulseEvaluator.mjs:74-162`):** On wire, `wake/heartbeat_pulse`, `wake/permission_granted`, and `wake/task_state_changed` emit bare UUIDs / `01H*` event IDs in `sourceEventIds` rather than `HEARTBEAT_PULSE*` or `PERMISSION*` prefixes, and `wake/task_state_changed` notifies a task's originator on terminal transition. `POST /wake/settle` will verify both the `MESSAGE:`/`TASK:` source-ID prefix whitelist **and** zero counts on `breakdown.{heartbeat_pulse,permission_granted,task_state_changed}` when `breakdown` is present, so non-message digests fail closed regardless of ID shape.
  2. **Native spool pointer release (`ai/daemons/wake/primaryWakeSpool.mjs:105`):** `isSpoolPointerActive` currently checks `state !== 'delivered'`. Settling a staged-but-uncommitted `normalPointer` or `urgentPointer` record to `skipped` would leave `isSpoolPointerActive === true` and block later wakes (`earlier native wake outstanding`). `isSpoolPointerActive` will treat any `TERMINAL_STATES` (`delivered`, `skipped`, `failed`, `unknown`) as inactive when `attempt?.recordKey !== entry.recordKey`, while `POST /wake/settle` protects any record referenced by `sessionState.inflight` or `sessionState.urgentInflight` (`recordKey` or `members`, whether `status` is `'unknown'` or unretired `'delivered'`) and triggers `drain()` when `skippedCount > 0`.

Origin Session ID: 59ca0d8c-dfdf-4d07-aefe-8ec5c658a398 🧭

- 2026-11-22T04:03:40Z @neo-gemini-maryam cross-referenced by PR #1895
- 2026-11-22T06:11:12Z @neo-gemini-maryam referenced in commit `c7bf5e7` - "fix(wake): move settlement off MCP response path and unify contract (#1890)"
- 2026-11-22T11:01:58Z @tobiu referenced in commit `a5f3482` - "feat(wake): skip pending wake digests whose sourceEventIds are handled (#1890) (#1895)

* feat(wake): skip pending wake digests whose sourceEventIds are handled (#1890)

* fix(wake): move settlement off MCP response path and unify contract (#1890)"
- 2026-11-22T11:01:59Z @tobiu closed this issue

