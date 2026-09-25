---
id: 512
title: 'The health surface projects the wake receiver''s dispatch outcome, so an armed-but-undeliverable seat is expressible'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-preview
createdAt: '2026-09-25T21:04:40Z'
updatedAt: '2026-09-25T22:09:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/512'
author: neo-preview
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
closedAt: '2026-09-25T22:09:45Z'
---
# The health surface projects the wake receiver's dispatch outcome, so an armed-but-undeliverable seat is expressible

## Context

The delivered half of #503, split out so closing it does not close a ticket whose other half is undelivered. #503 carries two independent defects under one number: the seat envelope writer's schema drift (half 1) and the absent delivery projection (half 2). They share a root shape but not an owner, a lane, or a merge — half 1 is the operator's seat-provisioning lane and needs a real dispatch to verify, half 2 is a substrate projection with no deployment step at all. One ticket forced them to land together or not at all.

## The Problem

Every wake surface in the substrate projects the seat's **intent** — `status: 'active'`, `armed: true`, `harnessTarget: 'a2a-webhook'` — and none projected the receiver's **outcome**. Those are different questions, and the difference is invisible until a wake stops landing.

A subscription can be unambiguously armed, correctly routed, correctly signed, and still fail **every** dispatch. On the measured host that combination read healthy on every surface while one seat accumulated 114 consecutive failures over nineteen days: the subscription list said `active`, `who_is_online` said present, the orchestrator digest carried no delivery observation, and healthcheck reported gate and heartbeat liveness — never an outcome. **No surface existed on which "delivered 0 of the last 114" could appear.**

The receiver had been persisting one complete record per dispatch the whole time, correctly, including an `outcomeReason` on every failure. The records were the substrate's own accurate account of what happened. Nothing read them.

The same conflation the sibling liveness defect names from the other direction — off, dead and blind arriving as one payload — but here the payload is *positive and wrong*: armed and deliverable, undeliverable in fact. An instrument that cannot be wrong is not evidence, and this one was confidently wrong for nineteen days.

## The Fix

**One projection, on the surfaces the fleet already reads.**

- A pure per-subscription delivery verdict derived from the receiver's records: `state` (`reachable` / `unreachable` / `unknown`), `consecutiveFailures`, `lastOutcomeReason`, `lastDeliveredAt`, `lastAttemptedAt`.
- Beside `subscription.armed` on the healthcheck wake block — **separately, not merged into it**, because `armed` already documented itself as the Memory-Core-side verdict that does *not* claim a wake will arrive. This adds the leg that can say otherwise.
- The same verdict available to `who_is_online`, so presence and reachability are separable there too.

**The rule is the loud direction, inherited rather than invented.** The status policy settled the *absent* case as `absent ⇒ active`, justified as failing loudly; that reasoning does not reach a row that is **present, active and unreachable**, because that row is unambiguous and unambiguous is not the same as reachable. So `reachable` is earned by an observed `delivered` and nothing else — no records, a still-pending dispatch, a malformed record, and an unreadable directory all read `unknown`. An **absent** records directory reports `deliveryReadable: true, deliveryReadReason: 'no-records'`; an **unreadable** one reports `deliveryReadable: false`. "Nothing was ever dispatched" and "everything is fine" must not read the same.

`skipped` counts against nothing and claims no delivery; `unknown` counts as a failure. Both are real populations on a live receiver (439 and 247 records when measured), so collapsing them together would either manufacture an alarm or hide one.

`state` + `consecutiveFailures` answer **now**; `lastDeliveredAt` + `lastOutcomeReason` answer **last known fact of each kind**, deliberately asymmetric with them — a recovered seat that forgets why it broke cannot be diagnosed.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| `wakeDeliveryProjection` (new, pure) | receiver record `state` + `outcomeReason` | per-subscription verdict; pure, no I/O | unreadable/pending/malformed ⇒ `unknown` | red-first arms: streak, recovery, skip-vs-unknown, ordering, malformed, per-subscription isolation |
| `wakeDeliveryReader` (new, I/O) | receiver records directory | reads + projects; never throws | absent ⇒ readable/`no-records`; unreadable ⇒ `deliveryReadable:false` | arms: real dir, absent, ENOTDIR, malformed, pinned convention |
| healthcheck wake block | `buildWakeFeaturesBlock` | gains `delivery` beside `subscription` | projection throws ⇒ `deliveryReadable:false` | non-vacuity control (AC-5) |
| `who_is_online` rows | `WakeSubscriptionService` row builder | present-and-reachable vs present-and-undeliverable | unreadable ⇒ `unknown` | open |

## Acceptance Criteria

- [x] **AC-3** A subscription whose dispatches fail carries `consecutiveFailures` ≥ 1 and the receiver's own `lastOutcomeReason` on the health surface. Red-first — the module did not exist when the spec was written.
- [x] **AC-5** *(non-vacuity control)* Neither permanently red nor permanently green: a delivered dispatch reads `reachable` with `consecutiveFailures: 0`, and a recovered seat's streak returns to 0. Without this arm a hardwired counter passes everything else in the file.
- [ ] **AC-4** `who_is_online` distinguishes present-and-reachable from present-and-undeliverable, and an unreadable delivery state reads `unknown`. The rows are built in `WakeSubscriptionService.mjs:807` (`_projectAgentLiveness`), which needs an async read plus a graph-node → `subscriptionId` mapping; `planeWhoIsOnlineReader` is only a passthrough, so the change belongs in the row builder.

## Avoided Traps

- **Deriving reachability from intent.** `status: active` answers "does this seat want wakes", never "can a wake land". Keeping them in one field is what made this invisible.
- **A counter with no non-vacuity control.** A streak that always reads 0 proves nothing; AC-5 exists so the fix cannot pass on a permanently-healthy payload.
- **Reading the log directory as the fix.** The records are correct and always were. The defect is that nothing projects them.
- **Reporting `unknown` as healthy.** That is the same conflation in a new coat, and it is the direction this ticket exists to prevent.
- **A test that reads the host's real records.** The unit suite pins the records directory to an absent path. Without it a developer run read the host's actual wake history inside `healthcheck()` — wrong, and slow (the memory-core + fleet sweep went 25.2s → 6.6s once isolated).

## Out of Scope

**Half 1 of #503** — the envelope writer's schema drift, the seat identity provisioning, and the shared contract declaration. That is the operator's seat-provisioning lane and needs a real dispatch to verify. This ticket does not touch it and its merge does not close it.

The receiver's state directory remains a `--state-dir` CLI flag with no config leaf, in a host-AgentOS root family outside the config plane-member system (those resolve under `<repo>/.neo-ai-data`), so the receiver state, the routes manifest and the courier spool all sit outside every declared config leaf. The reader resolves that root by the sibling courier convention plus a `NEO_WAKE_RECEIVER_RECORDS_DIR` override — testable and overridable, but **not** a declared contract. Reported, not papered over; unifying it is its own change.

## Live receipt

Run against the measured host's real records (8156 records, 10 subscriptions), the projection surfaced **two silently-undeliverable seats that no surface was reporting** — one carrying the identical envelope failure this work was filed about, on a seat never healed. Filed separately on @neo-fable's confirmation that neither route is his.

## Related

#503 (parent; half 1 remains open here) · PR #510 (the delivered AC-3 + AC-5) · the sibling liveness defect where off/dead/blind arrive as one payload.


## Timeline

- 2026-09-25T21:04:41Z @neo-preview added the `enhancement` label
- 2026-09-25T21:04:42Z @neo-preview added the `ai` label
- 2026-09-25T21:04:45Z @neo-preview assigned to @neo-preview
- 2026-09-25T21:05:01Z @neo-preview cross-referenced by PR #510
- 2026-09-25T21:05:36Z @neo-preview cross-referenced by #513
- 2026-09-25T21:05:47Z @neo-preview cross-referenced by #514
- 2026-09-25T21:55:29Z @neo-preview cross-referenced by #522
- 2026-09-25T22:09:46Z @tobiu closed this issue

