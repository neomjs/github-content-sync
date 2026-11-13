---
id: 1682
title: The wake receiver opens a fresh session on a signed rotation event
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T16:53:48Z'
updatedAt: '2026-11-13T19:20:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1682'
author: neo-opus-ada
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
blocking:
  - '[ ] 1383 On a sunset receipt the wake plane rotates the seat: receipt in, fresh session out'
closedAt: '2026-11-13T19:20:54Z'
---
# The wake receiver opens a fresh session on a signed rotation event

## Context

The operator wants a seat that writes its sunset handover to continue in a fresh session that starts with a first prompt, whether he is present or away. The wording is quoted in neomjs/neo-agent-institution#842.

The 2026-11-13 measurement ([#842 comment 6859174199](https://github.com/neomjs/neo-agent-institution/issues/842#issuecomment-6859174199)) found two things:
- The old loop (`checkSunsetted` → `SwarmHeartbeatService` → `resumeHarness`) cannot run on this plane.
- Of the processes that could type into a seat's app, only the host wake receiver holds the Accessibility grant.

Vega and I agreed a split (Tier 2.5, 16:47Z):
- #1383 owns the detector and the emitter.
- This ticket owns the receiver's final mile and the Claude seat's rotation route.
- #1383's installed receipt is `blocked_by` this ticket.

## The Problem

Nothing can open and submit a fresh session in the right seat's app today:
- The FM's `osascript` is refused with `-25211`.
- `resumeHarness` without a route activates `"Claude"` by name, and every Claude seat on this host shares that name.
- The receiver admits only `wake/digest`.
- Claude seats have no receiver route. Arming retires their typed routes at every session start, by design since #30.

Typing also carries a known hazard. #30 measured that `osascript` typing races a human at the keyboard. `-2700` is only the detectable branch; the undetectable one types into another window. A rotation that types a boot prompt must not repeat that.

## The Architectural Reality

- **Admission.** `ai/daemons/wake/receiver.mjs` admits a request through the `x-neo-wake-*` headers and an HMAC per route.
  - The envelope must match the route, and `eventType !== 'wake/digest'` answers `409 signed-route-mismatch` (`:508`).
  - Accepted records drain serially through `dispatchLocalWake` (`localWakeAdapters.mjs`).
- **Typing builders.** There are two.
  - The receiver's `buildOsascriptArgs` (`localWakeAdapters.mjs:784`) binds the target process by identity, preserves the user's draft and clipboard, and runs behind the dialog gate.
  - Its keystrokes land: Euclid's 11 deliveries from 14:04Z to 16:39Z on 11-13 were each followed by a turn of that seat. The receiver log holds one historical "not allowed to send keystrokes" line.
  - `resumeHarness.buildResumeOsascriptArgs` (`ai/scripts/lifecycle/resumeHarness.mjs:308`) is the old loop's copy and the only one that sends Cmd+N. Its unaddressed branch activates by app name.
- **Routes.** They come only from `a2a-webhook` records (`buildReceiverManifest.mjs`, `DELIVERABLE_HARNESS_TARGET`).
  - The GUI tuple comes from an explicit instance input, never from the record.
  - Composition is additive per identity.
- **Arming.** `armSeatWakePull` retires only `SENT_TO_ME` routes that type into a window (`typesIntoWindow` / the retire loop, `:104-131`). Other triggers are left alone.
- **Triggers.** `WakeSubscriptionService.validTriggers` (`:125`) is `SENT_TO_ME`, `TASK_STATE_CHANGED`, `PERMISSION_GRANTED`, `HEARTBEAT_PULSE`.
- **Route arming is the Fleet's.** At Start, `ai/services/fleet/armFleetSeatWake.mjs` arms a launched GUI seat's route, acting as the seat. It subscribes `SENT_TO_ME` `a2a-webhook` with `{adapter, appName, url: <fleet.wakeReceiverBase>/wake}` and publishes the route through `armSeatWakeRoute` with the launch `userDataDir`. `GUI_WAKE_DISPATCH` has no Claude Desktop entry, because the seat's own arming retires typed wake routes.
- **Matching is per trigger.** `wakeCoalescePolicy.mjs` matches `SENT_TO_ME`, `TASK_STATE_CHANGED` and `PERMISSION_GRANTED` by name, so a route on another trigger never receives a wake digest.
- **The old session retires on the first prompt.** In `wakeListenerHook.mjs`, a prompt claims the seat, and the previous owner's `Stop` listener exits `superseded`.
- **Human input is measurable without a grant.** `ioreg -c IOHIDSystem` reports `HIDIdleTime`.
- **Placement.** `npm run ai:structure-map -- --files --loc` puts the receiver, adapters and arming in `ai/daemons/wake/` (21 files). No new folder.

## The Fix

1. **Rotation route class, armed by the Fleet.**
   - Add `SESSION_ROTATE` to `validTriggers` and to the `manage_wake_subscription` trigger enum.
   - `armFleetSeatWake` gains a rotation dispatch for `claude-desktop`: `{adapter: 'osascript', appName: 'Claude', tabShortcut: '3'}`. At every Start it subscribes the seat's one `SESSION_ROTATE` `a2a-webhook` route to the same receiver URL and publishes it with the launch `userDataDir`.
   - `buildReceiverManifest` records which envelope the route admits: `session/rotate` for `SESSION_ROTATE`, `wake/digest` otherwise.
   - The `SENT_TO_ME` filter in `armSeatWakePull` leaves the route alone by construction.
2. **The seam (the envelope #1383 emits).** It uses the same frame and headers as a wake:

   ```
   schemaVersion: '1.0'
   eventType    : 'session/rotate'
   eventId      : 'session-rotate:<receipt message id>'
   subscriptionId, agentIdentity, emittedAt
   payload      : {receiptMessageId, oldSessionId, reason: 'sunset_restart', prompt}
   ```

   - The field beyond #1383's five is `subscriptionId`: the emitter addresses the seat's rotation route, never its pull route.
   - The receiver admits `session/rotate` only on a `SESSION_ROTATE` route and `wake/digest` only on the others. Both mismatches answer `409`.
3. **Final mile, in the receiver's own builder.**
   - `buildOsascriptArgs` gains a fresh-session step: Cmd+N after the tab shortcut, before the prompt clear.
   - A rotation record dispatches through `deliverOsascript` with that step and the payload's `prompt`, and only to an addressed instance. The default-target path is refused for rotation.
   - `resumeHarness`'s builder is not moved. It retires with the old loop (#1383).
4. **Human-input guard (#30).**
   - Before the first keystroke, a rotation reads `HIDIdleTime`. Under a named threshold (a constant beside `DIALOG_DEFER_BOUND`), the record defers to the receiver's existing 60 s re-drain and nothing is typed.
   - The defers are bounded and exhaust into a named failure, mirroring the dialog gate.
   - The frontmost assertion stays.
5. **Idempotency.** The record key is the receipt id. A re-emitted event for the same receipt is `duplicate`, never a second session.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `validTriggers` + the `manage_wake_subscription` `trigger` enum | `WakeSubscriptionService.mjs:125` | adds `SESSION_ROTATE` | unknown trigger refused, as today | tool handbook | unit |
| receiver admission | `receiver.mjs:508` | `session/rotate` only on `SESSION_ROTATE` routes; `wake/digest` elsewhere | `409 signed-route-mismatch` | receiver JSDoc | unit |
| rotation envelope | this ticket ↔ #1383 | frame above | none: the emitter matches it | receiver JSDoc, #1383 body | unit |
| Fleet seat arming | `armFleetSeatWake.mjs` (`GUI_WAKE_DISPATCH`) | arms one `SESSION_ROTATE` route per Claude Desktop Start with the launch tuple | `unarmed` with a reason; the Start never fails | module JSDoc | unit + installed |
| fresh-session step | `localWakeAdapters.mjs:784` (`buildOsascriptArgs`, extended) | Cmd+N step; addressed instance only; human-input guard | refuse / defer | adapter JSDoc | unit + installed |

**Decision Record impact:** `aligned-with` ADR 0002 §6.2.3. The server issues the signing key, and the manifest builder never mints one.

## Acceptance Criteria

- [ ] Unit: the receiver admits `session/rotate` only on a `SESSION_ROTATE` route. A `wake/digest` on that route answers `409`, and so does a `session/rotate` on a wake route.
- [ ] Unit: two events for one receipt id produce one dispatch; the second is `duplicate`.
- [ ] Unit: a rotation record without an addressed instance is refused before any `osascript` runs.
- [ ] Unit: the rotation builder sends Cmd+N after the tab shortcut and before the prompt clear. A wake's builder output is unchanged.
- [ ] Unit: `HIDIdleTime` under the threshold defers the record with no keystroke. Over it, the step runs. The defer bound exhausts into a named failure.
- [ ] Unit:
  - A Claude Desktop Start arms exactly one `SESSION_ROTATE` route with the launch `userDataDir` tuple, and a second Start reuses it.
  - Codex and Antigravity Starts arm no rotation route.
  - `armSeatWakePull` leaves the `SESSION_ROTATE` route in place and still retires typed `SENT_TO_ME` routes.
- [ ] Installed receipt (post-merge, local plane):
  - A signed `session/rotate` for one Claude seat opens and submits a fresh session in that seat's app. Until #1383's emitter ships, the event comes from a diagnostics rig beside `dialogGateRig.mjs` that reads only the target seat's route.
  - The pid list of every Claude instance is unchanged before and after.
  - The fresh session's first prompt carries the receipt id.
  - The old session's `Stop` listener exits `superseded`.

## Out of Scope

- **The detector, its guards and the emitter:** #1383 (Vega).
- **Codex and Antigravity seats:** their apps' fresh-session step is unmeasured. Each becomes one row in the same step once measured.
- **The FM's visible half:** neomjs/neo-agent-institution#842 Fix 2 and Fix 3.
- **The fleet-server receiver** (`fleetWakeReceiver.mjs`, installs without a checkout).
- **Retiring the old loop** (`checkSunsetted`'s sunset branch, `SwarmHeartbeatService` step 3, `resumeHarness` and its builder): that goes with #1383's detector, which replaces it.

## Avoided Traps

- **Enabling `swarmHeartbeatEnabled`.** The lane has no home: the host edge is graphless and pins it off, and the container does not own it. It would also type by app name.
- **A `SENT_TO_ME` typed route for Claude.** Arming retires it at every session start, deliberately (#30).
- **The FM as the actuator.** It has no Accessibility grant (`-25211`), and it would need a second container→host path.
- **Moving `resumeHarness`'s builder into the receiver.** That leaves two typing builders, and the receiver's is the hardened one: identity-bound target, draft restore, dialog gate.
- **Arming from the seat's SessionStart hook.** The hook does not know the receiver URL. `armFleetSeatWake` holds the credential, the launch tuple and the URL, and it runs for every harness.
- **`clear_session` on itself, plus a pulled wake.** A cleared, idle session runs no `Stop` hook, so nothing submits the first prompt.
- **The deep link `claude://code/new?q=`.** It only pre-fills the prompt, and URL routing across same-bundle instances is ambiguous.

## Related

- #1383: the detector and emitter; blocked by this ticket.
- neomjs/neo-agent-institution#842: the FM half and the measurement.
- #30: the typed-delivery hazard.
- #1681: a deaf listener after a failed turn.
- #571: the seat folder layout.
- #1625: operator presence.

**Intake** (self-authored, stage 2):
- Prescription checked against `ai/daemons/wake/localWakeAdapters.mjs`, which is a better owner than moving `resumeHarness`'s builder. Fix 3, its ledger row and its AC were corrected at 16:57Z.
- Prescription checked against `ai/services/fleet/armFleetSeatWake.mjs`, which is a better owner than the seat's SessionStart hook: the Fleet holds the seat's credential, its launch tuple and the receiver URL. Fix 1, the Architectural Reality, the ledger row and the AC were corrected in the next edit.

**Sweeps:**
- **Live latest-open + closed sweep:** the latest 20 open issues and a closed search on `fresh session rotate OR resume sunset receiver`, at 2026-11-13T16:51:04Z and re-checked at 16:53:12Z. #30's Out of Scope and Avoided Traps were read. No equivalent or prior exclusion.
- **Backlog:** 155 open (red). No owning ticket can take this: #1383 is a one-`Resolves` standalone that names this ticket as its sibling, and #842 keeps the FM half.
- **A2A claims** (last 30, all read states): none overlapping.
- **Memory Core** (problem nouns): no prior mechanism.
- **Own assignments** (#1681, #1531, #571, #133, #1508): none covers it.

Ownership: self-assigned. This is not a `good first issue`: #1383 waits on it, and its receipt needs a live multi-seat host with the receiver's grant.

Origin Session ID: cd254cdd-415d-46ed-a61f-022e90ac619c
Retrieval Hint: "wake receiver opens fresh session signed rotation event SESSION_ROTATE route human-input HIDIdleTime guard"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T16:53:49Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T16:53:50Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T16:53:50Z @neo-opus-ada added the `ai` label
- 2026-11-13T16:53:50Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T16:55:19Z @neo-opus-vega marked this issue as blocking #1383
- 2026-11-13T16:56:44Z @neo-opus-vega cross-referenced by #842
- 2026-11-13T16:56:46Z @neo-opus-vega cross-referenced by #1383
- 2026-11-13T17:16:51Z @neo-opus-ada cross-referenced by PR #1683
- 2026-11-13T17:50:23Z @neo-opus-ada referenced in commit `b7caf66` - "fix(wake): read idle at the keystroke, key rotations by receipt (#1682)

The human-input read now comes after the pid lookup and the dialog probe, right before the first keystroke, and a failed, killed or timed-out ioreg read counts as no observation. A session/rotate envelope is admitted only when its eventId is session-rotate:<payload.receiptMessageId>, so one receipt is one record and a re-labelled re-emission is refused."
- 2026-11-13T18:17:56Z @neo-opus-ada cross-referenced by #1681
- 2026-11-13T18:18:20Z @neo-opus-grace cross-referenced by #377
- 2026-11-13T18:57:22Z @neo-gpt-euclid cross-referenced by PR #378
- 2026-11-13T19:20:53Z @tobiu referenced in commit `a97a877` - "feat(wake): the receiver opens fresh sessions on rotation (#1682) (#1683)

* feat(wake): the receiver opens fresh sessions on rotation (#1682)

A seat that writes its sunset handover continues in a fresh session: a SESSION_ROTATE route, armed by the Fleet at Start with the launch userDataDir, admits only session/rotate envelopes. The host wake receiver opens the seat's fresh session with the app's shortcut (Cmd+N for Claude Desktop) and submits the first prompt, only to the seat's own instance and only after a quiet minute of keyboard and pointer. Wake routes, wake keystrokes and the context gate for wakes are unchanged.

* fix(wake): read idle at the keystroke, key rotations by receipt (#1682)

The human-input read now comes after the pid lookup and the dialog probe, right before the first keystroke, and a failed, killed or timed-out ioreg read counts as no observation. A session/rotate envelope is admitted only when its eventId is session-rotate:<payload.receiptMessageId>, so one receipt is one record and a re-labelled re-emission is refused."
- 2026-11-13T19:20:55Z @tobiu closed this issue

