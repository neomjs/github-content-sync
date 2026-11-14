---
id: 1692
title: The Fleet reads each seat's session rotations for its card and journal
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T22:54:54Z'
updatedAt: '2026-11-14T02:36:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1692'
author: neo-opus-ada
commentsCount: 4
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1383 On a sunset receipt the wake plane rotates the seat: receipt in, fresh session out'
blocking:
  - '[ ] 842 The Fleet Manager rotates one seat''s session on request, no other harness touched'
closedAt: '2026-11-14T02:36:57Z'
---
# The Fleet reads each seat's session rotations for its card and journal

## Context

neomjs/neo-agent-institution#842 keeps the Fleet Manager's visible half of a session rotation. The roster card and the journal carry each rotation, and the old session reads retired once a fresh one has taken over.

The rotation itself runs in the plane and on the host:
- #1383 (Vega, merged in #1693) detects the sunset receipt, emits `session/rotate` and keeps one `sessionRotation` record per receipt;
- the host wake receiver opens the fresh session (#1682, merged in #1683).

The Fleet reads none of this yet. This ticket is that read; the card and the journal rows stay on #842. The measurements behind it are in neomjs/neo-agent-institution#842 comment 6866729215. The record's shape is Vega's, agreed over A2A on 2026-11-13.

## The Problem

The operator reads his fleet from the roster card and the activity history, and a rotation shows in neither. That includes one that ran at night, one the plane refused inside its minimum interval, and one whose fresh session never came up.

## The Architectural Reality

- `FleetManager.armSeatWake` (`ai/services/fleet/FleetManager.mjs`) arms the seat's rotation route at Start and returns it as `rotationRoute`. The lifecycle record keeps only `wakeRoute` (`lifecycle.setWakeRoute`, taken only from the launch that armed it).
- #1383 stores its decision on the receipt's message node as `properties.sessionRotation`:
  - `state`: `requested`, `completed`, `refused` or `incomplete`, each stamped (`requestedAt`, `completedAt`, `refusedAt`, `incompleteAt`);
  - `reason`: for refused, `prompted-after-receipt`, `newer-session` or `minimum-interval` (with `nextEligibleAt`); for incomplete, `receipt-not-consumed`, `undelivered` or `no-fresh-session`;
  - `oldSessionId`, and `newSessionId` once completed. Both are Memory Core session ids.

  #1383 also adds `readSunsetReceipt` (`ai/daemons/wake/sunsetReceipt.mjs`). The projection onto the wake observation row moved into this ticket (agreed with Vega over A2A on 2026-11-13): `rotation: {state, reason, receiptId, decidedAt, oldSessionId, newSessionId, nextEligibleAt} | null`.
- That row is composed by `readActiveWakeSubscriptionObservations` (`ai/services/memory-core/readActiveWakeSubscriptionIdentities.mjs`) and served as `manage_wake_subscription` `{action: 'fleet-identities'}`. The Fleet reads it through `createPlaneWakeObservationsReader` (`ai/services/fleet/planeWakeIdentitiesReader.mjs`).
- `fleetWakeRoutes` (`ai/services/fleet/fleetWakeRoutesSource.mjs`) answers each axis of a seat's wake path as its own `{state, reason}`. It already reads that row for its subscription axis.
- `fleetActivity` (`ai/services/fleet/fleetActivityComposer.mjs`) composes injected adapters in two slots, `a2a` and `prLane`. It claims `wired` only when every slot is wired.
- `src/fleet/contract/cockpit.mjs` names the cockpit's sources and event types for the Institution client.

## The Fix

1. Record `rotationRoute` on the lifecycle record beside `wakeRoute`, from the arming launch only.
2. A `rotation` axis on `fleetWakeRoutes`, projected from the row's `rotation`:
   - each of the four states, with the record's reason;
   - `completed` names `oldSessionId` and `newSessionId`, which is the retired mark #842 renders;
   - `incomplete` with `no-fresh-session` is the report that the old session never handed over, and nothing is stopped;
   - a plane whose row carries no `rotation` reads `unobserved`, a seat without a record `none`, and an unreadable record `unknown` with its reason.
3. A rotation slot in `fleetActivity`: `fleetRotationActivityAdapter.mjs` beside the A2A and PR-lane adapters. One event per receipt and state (`eventId` `fleet:sessionRotation:<receiptId>:<state>`, from `createFleetCockpitEventId`), with `payload: {receiptId, state, reason, oldSessionId, newSessionId}`. The slot is optional: a composer without a rotation reader does not compose it.
4. `cockpit.mjs` gains the rotation source and the rotation event type.
5. The plane projection in `readActiveWakeSubscriptionObservations`: each observation carries `rotation`, the decision on the seat's newest sunset receipt from the last day, for the seats holding an active `SESSION_ROTATE` route. `decidedAt` is the current state's stamp. A seat without a route or a decision has `null`, and a receipt that cannot be read gives `{state: null}` for its seat alone.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Lifecycle record `rotationRoute` | this ticket | recorded beside `wakeRoute`, from the arming launch | a Start canceled by Stop records none | `armSeatWake` JSDoc | unit arm |
| Wake observation row `rotation` (`fleet-identities`) | this ticket; the record is #1383's | the decision on an armed seat's newest receipt within a day, stamped `decidedAt` | `null` for none; `{state: null}` for an unreadable receipt; the cache-double path carries none | `readActiveWakeSubscriptionIdentities.mjs` and `fleetIdentities` JSDoc | unit arms over a real graph store |
| `fleetWakeRoutes` row: `rotation` axis | this ticket; the record is #1383's | the four states with reasons and both session ids | `unobserved` when the plane carries no `rotation`; `none` without a record; `unknown` with a reason when the record is unreadable | `fleetWakeRoutesSource.mjs` module docs | unit arms |
| `fleetActivity`: rotation slot and events | this ticket | one event per receipt and state | an unreadable slot keeps the composite from claiming `wired` | composer and adapter JSDoc | unit arms |
| `cockpit.mjs`: rotation source and event type | this ticket, the Institution client | new identifiers; none renamed | none | module JSDoc | the client's import in neomjs/neo-agent-institution#842 |

## Acceptance Criteria

- [ ] AC-1: after a Start the lifecycle record carries `rotationRoute`; a Start canceled by Stop records none.
- [ ] AC-2: the `rotation` axis answers each of the four states with its reason, and `completed` with both session ids. A plane that carries no `rotation` answers `unobserved`, a seat without a record `none`, and an unreadable record `unknown`. One unit arm each, over an injected observation reader.
- [ ] AC-3: `fleetActivity` emits one rotation event per receipt and state, a re-read emits no duplicate, and an unreadable rotation slot keeps the composite from `wired`.
- [ ] AC-4: `cockpit.mjs` names the rotation source and event type, and no existing identifier changes.
- [ ] AC-5: the wake observation row carries each armed seat's rotation from its newest receipt within a day, `null` for an unarmed seat, an undecided receipt or a day-old one, and `{state: null}` for an unreadable receipt without touching the other seats' rows (unit arms over a real graph store).

## Out of Scope

- The card, the journal rows and their screenshots (neomjs/neo-agent-institution#842).
- Deciding, emitting and recording rotations (#1383).
- The receiver's step (#1682).
- Raising a seat's window: #842's open question to the operator.
- Whether `fleetHistory` keeps past rotation events (comment 6867321459).

## Avoided Traps

- **Retiring on the receiver's delivery.** Delivery means the prompt was typed and submitted, not that the fresh session took over. #1383's `completed` waits for the fresh session's first memory.
- **Matching the record's session ids against the listener record.** `oldSessionId` and `newSessionId` are Memory Core ids. The listener record keys owners by the harness `session_id`, which the plane never sees.
- **A second read for the record.** The observation row the Fleet already reads carries it, and a new tool would add a transport for one field.
- **Reading every seat's receipts on each roster read.** Only the seats armed for rotation are read, over the one-day `sentAt` range the message index serves.

## Related

neomjs/neo-agent-institution#842 renders this. #1383 records it and blocked this until #1693 merged. #1682 is the receiver. #1508 is the Fleet Manager epic; its sub count is past the cap, so this is not linked as a sub.

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 open Brain issues (created-descending) and closed searches for "fleet rotation activity wake routes axis" and "rotation session fleet", read at 2026-11-13T22:54Z. Brain #1383 is the plane side this consumes; no Fleet-side equivalent or prior exclusion. Backlog band: red (153 open). The owning outcome is neomjs/neo-agent-institution#842, and a Brain PR needs a Brain ticket.
A2A in-flight sweep: `[lane-claim]` messages of the last day; only Vega's #1383 (plane side, complementary). Memory Core sweep: no prior Fleet-side rotation read. Own-assignment sweep: none overlapping.
Structure map: `npm run ai:structure-map -- --files --loc` exits 0. The adapter sits beside `fleetA2AActivityAdapter.mjs` and `fleetPrLaneActivityAdapter.mjs` in `ai/services/fleet`.

Origin Session ID: a1330b6a-6d76-4638-ae19-cb6a20403c70
Retrieval Hint: "fleet reads session rotation record: lifecycle rotationRoute, fleetWakeRoutes rotation axis, fleetActivity rotation slot, wake observation rotation projection"


## Timeline

- 2026-11-13T22:54:54Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T22:54:55Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T22:54:56Z @neo-opus-ada added the `ai` label
- 2026-11-13T22:54:56Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T22:55:16Z @neo-opus-ada marked this issue as being blocked by #1383
- 2026-11-13T22:55:24Z @neo-opus-ada marked this issue as blocking #842
- 2026-11-13T23:00:38Z @neo-opus-ada cross-referenced by #842
### @neo-opus-ada - 2026-11-13T23:00:56Z

**Intake (author, same session): the stage-2 prescription check.**

- `Prescription checked: ai/services/fleet/FleetManager.mjs (armSeatWake) — owns the concern.` It already arms the rotation route and is the one writer of the launch-guarded `wakeRoute`; `rotationRoute` belongs beside it.
- `Prescription checked: ai/services/fleet/fleetWakeRoutesSource.mjs — owns the concern.` It already reads the wake observation row for its subscription axis. A rotation is a separate concern from subscription intent, so it gets its own axis rather than a field on that one.
- `Prescription checked: ai/services/fleet/fleetActivityComposer.mjs — owns the concern.` It is the Fleet's only activity producer, with injected per-source adapters, so the rotation adapter goes beside the A2A and PR-lane ones.
- `Prescription checked: src/fleet/contract/cockpit.mjs — owns the concern` (shared identifiers).

**Scope note, 2026-11-14:** the operator ruled on neomjs/neo-agent-institution#842 that the card needs no raise or open action; the FM updates on its own. That is what this ticket feeds.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T23:12:33Z @neo-opus-ada referenced in commit `b5a25db` - "feat(fleet): the wake-routes read answers each seat's rotation (#1692)"
### @neo-opus-ada - 2026-11-13T23:15:36Z

**Progress and the AC-3 shape (Ada).** Branch `ada/1692-fleet-rotation-read`, no PR yet.

**Done, each red on `dev` first:**
- b5ddbe1f, AC-1: `setRotationRoute` beside `setWakeRoute`. A `launchRecord()` guard now serves `setWakeRoute`, `setRepoOutcomes` and `setRotationRoute`. `status()` projects `rotationRoute` through an allowlist (no signing key), and `armSeatWake` records both routes from the arming launch only.
- b5a25db1, AC-2 (Fleet side): `fleetWakeRoutes` seats carry `rotation`, from the same observation scan as the subscription axis. The four record states project verbatim with both Memory Core session ids. A row without the field reads `unobserved`, `rotation: null` reads `none`, an out-of-contract state reads `unknown`, and an unreadable scan gives `unknown` with the subscription's reason. `createPlaneWakeObservationsReader` carries `rotation` only when the plane names it.

**Still open:**
1. **The plane projection** (decided with Vega: it is not in Brain #1693). Once #1693 is on `dev`, `readActiveWakeSubscriptionObservations` calls `readSunsetReceipt` (`ai/daemons/wake/sunsetReceipt.mjs`) per identity. It uses a bounded `sinceIso` held as a module constant, not a config leaf, and projects `receipt.sessionRotation` as `{state, reason, receiptId, decidedAt, oldSessionId, newSessionId, nextEligibleAt}`, or `null` without a record. Check that `WakeSubscriptionService.fleetIdentities` passes the field through.
2. **AC-3, the shape I propose:**
   - A third composer slot, `rotation`. Its reader takes the wake-routes snapshot (`readWakeRoutes`, injected through `wireFleetActivityReadSource` from the fleet-server entry), so the projection keeps one authority (`rotationRow`) and the plane is not called twice.
   - `fleetRotationActivityAdapter.mjs` maps each seat whose `rotation` holds a record to one event: `eventId` `rotation:<receiptId>:<state>`, `type` `session-rotation`, `occurredAt` = `decidedAt`, and `payload` `{receiptId, state, reason, oldSessionId, newSessionId, nextEligibleAt}`.
   - Its capability: `degraded` when every seat reads `unobserved` (the plane predates the field) or the subscription axis is unknown; otherwise `wired`, possibly with zero events.
   - The composer's unanimity rule then marks `fleetActivity` degraded wherever rotations are unobserved. Plane and Fleet ship in the same cut, so that happens only on a mixed deployment, which is the honest reading there.
   - **Open:** whether `fleetHistory` persists past events. A seat's row carries only its newest receipt's decision, so a later receipt drops the earlier `completed` from live snapshots.
3. **AC-4:** `cockpit.mjs` gains the rotation source and the `session-rotation` event type, together with AC-3 so neither lands unused.
4. Then the PR: a GPT reviewer at green.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T23:26:19Z @neo-opus-ada referenced in commit `95634e6` - "feat(fleet): the activity history carries each seat's session rotation (#1692)"
### @neo-opus-ada - 2026-11-13T23:27:17Z

**Handover (Ada, solo-refresh sunset; my next session keeps ownership).** Branch `ada/1692-fleet-rotation-read` at 95634e6a, three commits, no PR yet. Each commit's new arms were red on `dev` first. The fleet services folder passes 2315, with 3 skipped.

- **b5ddbe1f, AC-1:** `rotationRoute` on the lifecycle record, launch-guarded. `status()` projects it through an allowlist.
- **b5a25db1, AC-2 (Fleet side):** the `rotation` axis on `fleetWakeRoutes`. The plane reader carries the field only when the plane names it.
- **95634e6a, AC-3 + AC-4:**
  - `fleetRotationActivityAdapter.mjs` emits one `session-rotation` event per seat with a record. Its id comes from `createFleetCockpitEventId`: `fleet:sessionRotation:<receiptId>:<state>`.
  - The slot reads `degraded` when every seat reads `unobserved` or a seat reads `unknown`.
  - The composer gains an optional third slot, `readRotationSnapshot`; without a reader it is not composed. `wireFleetActivityReadSource` takes `readWakeRoutes`, and `devFleetServer` passes it the wake-routes source's reader.
  - `cockpit.mjs` gains `rotation: 'fleet:sessionRotation'` and `session-rotation`.

**Pickup:**
1. **When Brain #1693 is on `dev`:** rebase this branch onto it. It has no PR yet, so the rebase is free. Then add the plane projection in `readActiveWakeSubscriptionObservations`:
   - call `readSunsetReceipt(db, identity, {sinceIso})` from `ai/daemons/wake/sunsetReceipt.mjs`, with a bounded `sinceIso` held as a module constant;
   - set `rotation` to `{state, reason, receiptId: receipt.id, decidedAt, oldSessionId, newSessionId, nextEligibleAt}` from `receipt.sessionRotation`, or `null` without one. `decidedAt` is the stamp of the current state (`requestedAt`, `completedAt`, `refusedAt` or `incompleteAt`);
   - check `hostBarrelImportReach.spec.mjs`, and that `WakeSubscriptionService.fleetIdentities` passes the field.
2. **Open the PR** (`Resolves #1692`) and request a GPT reviewer at green. Name two deltas: the event id format, and the slot being optional.
3. **Open:** whether `fleetHistory` keeps past rotation events (comment 6867321459).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-14T00:54:55Z @neo-opus-ada cross-referenced by PR #1694
- 2026-11-14T01:03:39Z @neo-opus-ada referenced in commit `98c3c1d` - "test(memory-core): the fleet read's redaction pin admits the rotation decision (#1692)

The fleet-identities disclosure test pinned the observation to identity, lastPollAt and pullRoute. The rotation decision joins the fleet-disclosure class by design: state, reason, the receipt and Memory Core session ids, and stamps, never route material or message content. The pin now names it and checks it stays null for unarmed seats; the module doc states the class."
### @neo-opus-ada - 2026-11-14T01:35:30Z

**Handover (Ada, solo-refresh sunset at the operator's yes; my next session keeps ownership).**

- **PR #1694** at `98c3c1de`, all six required checks green, Euclid requested. Six commits on `ada/1692-rotation-projection`, rebased onto #1693.
- **AC-5, the plane projection,** is in `readActiveWakeSubscriptionObservations`:
  - armed seats only (an active `SESSION_ROTATE` route), with a one-day receipt window;
  - `decidedAt` is the current state's stamp;
  - an unreadable receipt gives `{state: null}` for its seat alone.
- **The `fleet-identities` disclosure pin** now names `rotation`; the PR asks the reviewer to check that class.
- **This body** was corrected to the shipped contract: the three plane answers, the event id `fleet:sessionRotation:<receiptId>:<state>`, and AC-5 with its ledger row.
- **Pickup:** answer Euclid's review against `98c3c1de`, then the handoff at approval. After merge, neomjs/neo-agent-institution#842 renders the card line and the journal rows.
- **Cleanup:** `ada/1692-fleet-rotation-read` holds the pre-rebase commits (auto mode refused the force push). Delete it after merge.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-14T02:09:33Z @neo-opus-ada referenced in commit `5ed59d8` - "fix(fleet): the rotation slot keeps an unreadable roster degraded (#1692)

A roster the wake-routes source cannot read answers seats: [] with
confidence 'none'. The adapter kept only the array, so the slot read
wired/observed and a healthy pair of other slots made the composite
look wired. It now degrades with the source's reason; an empty roster
(confidence 'partial') stays quiet. Euclid's RA-1 on #1694."
- 2026-11-14T02:36:57Z @tobiu referenced in commit `fef1702` - "feat(fleet): the Fleet reads each seat's session rotation (#1692) (#1694)

* feat(fleet): the lifecycle record keeps the seat's rotation route (#1692)

* feat(fleet): the wake-routes read answers each seat's rotation (#1692)

* feat(fleet): the activity history carries each seat's session rotation (#1692)

* feat(fleet): the plane projects each armed seat's rotation onto the wake read (#1692)

readActiveWakeSubscriptionObservations adds rotation to each observation: the decision on the seat's newest sunset receipt within a day, for seats holding an active SESSION_ROTATE route, stamped decidedAt by its own state; null for none, and a record of no known state when a seat's receipt cannot be read, so one bad receipt never darkens the wake axis.

* docs(fleet): the rotation comments name the code, not the ticket (#1692)

* test(memory-core): the fleet read's redaction pin admits the rotation decision (#1692)

The fleet-identities disclosure test pinned the observation to identity, lastPollAt and pullRoute. The rotation decision joins the fleet-disclosure class by design: state, reason, the receipt and Memory Core session ids, and stamps, never route material or message content. The pin now names it and checks it stays null for unarmed seats; the module doc states the class.

* fix(fleet): the rotation slot keeps an unreadable roster degraded (#1692)

A roster the wake-routes source cannot read answers seats: [] with
confidence 'none'. The adapter kept only the array, so the slot read
wired/observed and a healthy pair of other slots made the composite
look wired. It now degrades with the source's reason; an empty roster
(confidence 'partial') stays quiet. Euclid's RA-1 on #1694."
- 2026-11-14T02:36:58Z @tobiu closed this issue

