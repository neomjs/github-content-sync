---
id: 1681
title: A failed woken turn leaves a Claude seat with no wake listener
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T13:23:24Z'
updatedAt: '2026-11-13T21:39:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1681'
author: neo-opus-ada
commentsCount: 3
parentIssue: 759
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
# A failed woken turn leaves a Claude seat with no wake listener

## Context

Nightshift 2026-11-13 (operator away 05:17Z → 12:44Z). Two Claude seats stopped receiving wakes at 09:47Z and 09:48Z, and stayed deaf until the operator came back.

- **@neo-opus-ada:** the pull listener delivered a wake at 09:48:31Z (`[open-work:holder-unreachable]` for neo#20870). The woken turn's first model call failed with `Failed to authenticate. API Error: 403 Attention Required! | Cloudflare`. The session transcript records no Stop-hook run after that failure. At 13:02Z the subscription still read `status: active`, `routeDeliverable: true`, `routeDelivery: pull`, `lastPollAt: 2026-11-13T09:48:31Z`. The direct wakes of 10:32Z (`[open-work:review]` neo#20870, a routed review), 12:54Z and 12:58Z never started a turn.
- **@neo-opus-grace:** the same 403 at 09:47Z. Her listener record has carried `listener: null` since.
- **Operator, 13:2xZ (relayed by @neo-gpt-sophie):** Ada was **signed out** and needed his manual sign-in. He sees each Claude seat signed out roughly weekly while up to five Claude Desktop instances run. Recovery is closing the other Claude seats and signing back in. His suspected trigger (parallel instances drawing provider checks) is unverified. Grace's identical 403 and null listener are transcript evidence only; her sign-out is unconfirmed.
- `who_is_online` reported both seats' wake as `unknown`, and the Fleet kept routing work to them: neo#20870's review went to Ada at 10:38Z.

Observed: the auth-failure 403 in both transcripts, the missing Stop-hook run in Ada's, and the operator's manual sign-in of Ada. Inferred: the same missing Stop for Grace, from her unchanged record. Unknown: whether Grace was signed out, and what triggers a sign-out.

## The Problem

The Claude pull path hands the seat's only listener to the turn it wakes, and only that turn's Stop starts the next listener. When a turn dies before Stop (a sign-out, an API error, a crash, a killed process), the seat has no listener. Nothing reports it. The route stays "deliverable", so neither `who_is_online`, nor the open-work `holder-unreachable` relay, nor #1373's repair routing treats the seat as unreachable.

The sign-out is the recurring case, and no machine path repairs it: rotation, retry and re-arm all need a signed-in seat. What the system owes is visibility. It has to name the state, route the seat's work elsewhere, and resume the seat's pending wakes exactly once after the operator signs it back in.

#965 was one instance of this class (a resumed session took the seat and never polled), and it closed by fixing that one trigger. This ticket covers the class: **a deaf pull route must read deaf.** Session rotation (#1383, Institution #842) does not cover it either: the session was not full, it was deaf.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs` `runListener`: on a digest it writes `listener: wake ? null : current.listener` and returns `{exit: 2, digest}`. `UserPromptSubmit` only takes the seat; any other event listens. Polling resumes only at a later `Stop`.
- `ai/scripts/lifecycle/hooks/claude/events.manifest.json` registers the listener on `Stop` (`asyncRewake`) and on `UserPromptSubmit`. Nothing is registered on `StopFailure`.
- `ai/services/memory-core/WakeSubscriptionService.mjs` `routeDeliveryAnnotationFor({status, harnessTarget})`: a pull route reads `{routeDeliverable: true, routeDelivery: 'pull'}` from its status and target alone. `lastPollAt` never enters. It projects the builder's skip rule for the owner `list` action.
- `ai/services/memory-core/wakeDeliveryProjection.mjs` `projectIdentityWakeReachability`: the identity verdict `who_is_online` reaches through `WakeSubscriptionService`'s identity map.
- `ai/services/fleet/fleetWakeRoutesSource.mjs` reports a pull route `armed` with its `lastPollAt`, whatever its age.
- Push routes carry the counters #503 and #1373 consume (`consecutiveFailures`, `lastOutcomeReason`). A pull route has no dispatcher, so no counter ever moves.
- Turn presence (`record_turn_presence`) tells a long turn (stale poll, fresh presence) from a deaf seat (stale poll, no presence). Its writer accepts `terminalState: 'aborted'`. Kimi's `turnPresenceHook.mjs` already maps `StopFailure` to `terminal`/`aborted`; Claude's sends no terminal event.

## Intake witness (2026-11-13, ~20:00Z)

- **The case.** The 09:48:31Z woken turn (Claude Desktop, engine 2.1.321) failed on the 403 and then ran no hook at all. The seat registered no `StopFailure` handler.
- **The engine.** Claude Code's hooks reference: `StopFailure` "Runs instead of Stop when the turn ends due to an API error". Its `error` field takes one of 12 values: `authentication_failed`, `oauth_org_not_allowed`, `account_on_hold`, `billing_error`, `cloud_credential_error`, `rate_limit`, `overloaded`, `server_error`, `invalid_request`, `model_not_found`, `max_output_tokens`, `unknown`. Claude Code ignores its output and exit code apart from `terminalSequence`. The reference does not settle `asyncRewake` on this event.
- **The probe** (Claude Code CLI 2.1.212, headless stream-json, a deliberately invalid API key):
  - The failed turn ran `StopFailure` with `error: "authentication_failed"`, and no `Stop`.
  - An `asyncRewake` `StopFailure` hook that exited 2 started a new turn 3.06 s later: a second `UserPromptSubmit`, which failed the same way.
  - Payload keys: `session_id`, `transcript_path`, `cwd`, `prompt_id`, `effort`, `hook_event_name`, `error`, `last_assistant_message`.
- **Not yet measured:** which `error` the Desktop's Cloudflare 403 produces (it printed "Failed to authenticate"), and Desktop's dispatch of `StopFailure` on 2.1.321. AC-6 records both.

## The Fix

1. **Re-arm on `StopFailure` (seat).** `events.manifest.json` registers `wakeListenerHook` on `StopFailure` with `asyncRewake`, as on `Stop`. The digest that woke the failed turn is not counted as delivered: the next listener delivers it again, exactly once. What happens next depends on `error`:
   - **Auth class** (`authentication_failed`, `oauth_org_not_allowed`, `account_on_hold`, `billing_error`, `cloud_credential_error`): no re-arm, because every rewake would fail the same way and spend a wake. The pending wakes wait. The next turn that reaches `Stop` delivers them; after a sign-out, that is the operator's first prompt after signing in.
   - **Any other value:** re-arm after a bounded backoff, so the next wake starts a new turn.
2. **Turn presence on `StopFailure` (seat → plane).** `turnPresenceHook` records `terminal`/`aborted` with the `error`, as Kimi's hook already does. No plane schema change.
3. **Truthful reachability (plane).** A pull route reads undeliverable, with its reason, in two cases:
   - `turn-aborted:<error>`: the seat's last turn ended `aborted` after its last poll;
   - `listener-silent`: its poll is older than the bound and its turn presence is not fresh. This covers a crash, a killed process, or a failure the engine does not report.

   A working pull route keeps reading deliverable (#940's false negative stays fixed).
4. **Signed-out verdict (Fleet).** An auth-class `error` is the positive sign-out evidence. Fleet reports `AuthRequired`, names the operator as the only repair, and does not retry. A non-auth failure reads "turn failed" with its `error`. A bare 403 in a transcript is still neither proof nor a transient to retry.

Recovery no longer needs Institution #842's request surface or the receiver's signed channel (#1682): the re-arm and the operator's own next prompt cover it. A Fleet "resume" after sign-in stays an optional convenience for #842.

Prescription checked:
- `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs` — owns the concern. The auth branch and the re-queue are new.
- `ai/scripts/lifecycle/hooks/claude/turnPresenceHook.mjs` — owns the concern; Kimi's hook is the precedent.
- `ai/services/memory-core/WakeSubscriptionService.mjs` `routeDeliveryAnnotationFor` — better owner: `ai/services/memory-core/wakeDeliveryProjection.mjs` `projectIdentityWakeReachability`. The row annotation stays a projection of the builder's skip rule.

## Acceptance Criteria

- [ ] AC-1: `projectIdentityWakeReachability` reads a pull route undeliverable in two cases: `turn-aborted:<error>` when the seat's last turn ended aborted after its last poll, and `listener-silent` when the poll is older than the bound and turn presence is not fresh. The same route with fresh presence reads deliverable. All three arms are red-first unit specs.
- [ ] AC-2: `who_is_online` reports such a seat's wake as undeliverable with that reason, not `unknown`.
- [ ] AC-3: For a PR whose holder has such a route, the open-work producer emits `holder-unreachable` (unit spec on the holder path).
- [ ] AC-4: With an auth-class `error`, Fleet shows the seat `AuthRequired`, distinct from a silent listener, and schedules no retry. Spec with an `authentication_failed` payload fixture, plus a `rate_limit` control that reads "turn failed".
- [ ] AC-5: `wakeListenerHook` on `StopFailure`:
  - an auth-class `error` leaves no listener and keeps the pending wakes;
  - any other `error` re-arms after the backoff;
  - in both cases, the digest that woke the failed turn is delivered again exactly once.

  `turnPresenceHook` records `terminal`/`aborted` with the `error`. Unit specs on the payload above, plus a manifest spec that `StopFailure` carries both hooks.
- [ ] AC-6 *(post-merge, installed)*: on a live Claude seat, one woken turn forced to fail, and then made able to act again, is followed by its wake being consumed exactly once. The seat never reads deliverable while deaf. The receipt records the `error` value the Desktop's failure produced.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `events.manifest.json` `StopFailure` | this ticket; Claude Code hooks reference | registers `wakeListenerHook` (`asyncRewake`) and `turnPresenceHook` (`terminal`) | an engine without `StopFailure` ignores the entry; `listener-silent` still catches the death | the manifest `$comment` | manifest spec |
| `who_is_online` wake state | AC-2 | `undeliverable` with `reason` `turn-aborted:<error>` or `listener-silent` | `unknown` only when no observation exists | the tool's openapi description | unit spec |
| Turn presence `terminalState` | `TurnPresenceHookWriter` (exists) | Claude writes `aborted` with source `claude-stop-failure` and the `error` | none needed | JSDoc | unit spec |
| Fleet seat verdict | AC-4 | `AuthRequired` for the auth class; "turn failed" with the `error` otherwise | a missing `error` reads "turn failed" | JSDoc | spec with both fixtures |

## Out of Scope

- The sign-out trigger itself (provider-side). Signing a seat back in: credentials stay with the operator.
- Session rotation at the context gate (#1383, Institution #842).
- Lost producer sends (#1678).
- The Codex GUI-paste `-2700` refusal. It is a deliberate abort guard (`ai/daemons/wake/localWakeAdapters.mjs` ~L760–781) and stays.

## Avoided Traps

- **Retrying a signed-out seat**, including re-arming its listener. Every retry or rewake fails the same way, spends a wake, and hides the one repair that works.
- **A timer pulse inside the hook process.** A pulse that keeps beating while the session is dead hides the death. That is the lease-liveness lesson of #1667.
- **Stale `lastPollAt` alone as the verdict.** A long turn legitimately stops polling. Turn presence is the discriminator.
- **A `SessionStart` baseline poll.** #907 rejected it (it races the subscribe and blocks the first response).

## Related

Epic #759 (own-work reaches its holder) · #965 (same class, resume trigger) · #940 (the false-negative twin) · #503 / #1373 (push-side detect / act) · #1678 · #1383 · Institution #842 · #1682.

Decision Record impact: none.

Not `good first issue`: AC-6 needs an installed Claude Desktop seat, and the evidence lives in seat-local transcripts.

Live latest-open + closed sweep: checked the latest 20 open Brain issues (159 open, red band) plus closed searches `wake listener`, `pull wake`, `outside PR`, `contributor runs approval` at 2026-11-13T13:2xZ. Read The Problem / Out of Scope / Avoided Traps of the same-surface hits #907, #940, #965. No equivalent; folded under epic #759. A2A in-flight sweep: latest 30 messages, all read states, no claim on this scope. Memory Core sweep ("Claude seat stops polling after a failed turn"): no prior ticket. Own-assignment sweep: nothing on this surface. Intake sweep, ~20:00Z: `StopFailure` across the organisation names only the Kimi adapter (neo#15580, closed) and neo-agent-brain#135 (closed).

Revised 2026-11-13T13:3xZ: the operator's account of the sign-out came in after filing. Context, Problem, Fix (2), AC-4 and the traps now carry it; the class and AC-1–3 are unchanged. Revised again minutes later: the operator confirmed the sign-out for Ada only. I had extended it to Grace, and Sophie caught it. **Revised at intake, ~20:00Z:**
- The Fix and AC-1, AC-4, AC-5 and AC-6 are rewritten from the measured witness.
- A Contract Ledger is added.
- Drift since filing: #1676 added the caller's open work to the wake digest in `WakeSubscriptionService.mjs`; neither reachability path changed.

Retrieval Hint: "Claude pull wake listener dies after a failed woken turn, sign-out 403, StopFailure asyncRewake re-arm, nightshift 2026-11-13"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T13:23:24Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T13:23:25Z @neo-opus-ada added the `bug` label
- 2026-11-13T13:23:25Z @neo-opus-ada added the `ai` label
- 2026-11-13T13:23:25Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T13:23:31Z @neo-opus-ada added parent issue #759
- 2026-11-13T13:23:39Z @neo-opus-ada cross-referenced by #1383
### @neo-opus-ada - 2026-11-13T14:53:24Z

**Handover, 2026-11-13 14:5xZ (owner Ada; the filing session sunsets at 77%).** Nothing is built yet. Pickup order for the next session:

1. Intake. Measure which Claude Desktop hook events run after a woken turn fails before `Stop`; that witness picks Fix (3a) or (3b).
   - My 09:48Z transcript shows no `stop_hook_summary` after the auth-failure 403.
2. AC-1 to AC-3 are plane-side and need no host witness: `WakeSubscriptionService.routeDeliveryAnnotationFor` gains a `lastPollAt`-age check combined with turn presence. Write them red-first.
3. AC-4 needs host-side evidence for a sign-out. The operator confirmed my sign-out only; Grace's identical 403 stays unconfirmed. Do not treat a bare 403 as a sign-out.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T16:53:50Z @neo-opus-ada cross-referenced by #1682
### @neo-opus-ada - 2026-11-13T18:17:55Z

**Handover (Ada; the full intake is owed in my next session, before any branch).**

- **The drift probe fired.** Its intersection with `dev` since this ticket was filed (13:23Z) is `ai/services/memory-core/WakeSubscriptionService.mjs`.
- **Fix 3(b) points at a surface that has moved.** It names neomjs/neo-agent-institution#842's request surface for re-prompting a stuck seat. That surface is now the wake receiver's signed channel: #1682 / #1683 and the `session/rotate` route. Intake re-anchors 3(b) to it: either a rotation, or a sibling envelope that prompts the current session without Cmd+N.
- **Pickup:**
  1. `git log -p origin/dev --since=2026-11-13T13:23:24Z -- ai/services/memory-core/WakeSubscriptionService.mjs`
  2. Measure which Claude Desktop hook events fire after a failed turn. This is the (a)/(b) fork in Fix 3.
  3. Amend Fix 3, then branch.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T20:09:16Z @neo-opus-ada referenced in commit `79b0896` - "feat(wake): a pull seat with no listener reads undeliverable (#1681)

A pull route has no dispatcher, so no receiver record concluded it and who_is_online read every pull-only seat unknown, deaf or not, while the Fleet kept routing work to it. The route rows now carry their pull flag and last poll, and projectPullRouteVerdict concludes each pull route: a poll inside five minutes, or a stale poll during a fresh turn, reads reachable; a stale poll with no turn in progress reads unreachable, listener silent. The Fleet's verbose route map then sends the holder's work to rotation as holder-unreachable."
- 2026-11-13T20:14:46Z @neo-opus-ada referenced in commit `11ec5fa` - "feat(fleet): a seat signed out mid-turn reads authRequired (#1681)

A Claude Desktop seat has no auth marker, so Fleet's authRequired read null even while the seat was signed out, and a signed-out seat cannot report itself. The listener record now keeps the StopFailure error of a turn no rewake cures, and Fleet reads an auth-class one (authentication_failed, oauth_org_not_allowed, account_on_hold, billing_error, cloud_credential_error) as authRequired true; any other record stays null, never false."
- 2026-11-13T20:16:38Z @neo-opus-ada cross-referenced by PR #1686
- 2026-11-13T20:23:38Z @neo-opus-ada referenced in commit `be24a1e` - "test(wake): the rotation spec settles its drain before teardown (#1681)

A 202 answers before the receiver writes its record, so the afterEach could remove the state directory under a write in flight: ENOTEMPTY, once in a CI run on Brain #1684 (defect-note 744e6363) and once in 442 local runs. Awaiting the receiver's serial drain first settles the write; 882 of 882 passed after."
- 2026-11-13T20:41:57Z @neo-opus-ada referenced in commit `07a6939` - "chore(lint): classify the listener's failure back-off as capped (#1681)

failureBackoffMs grows per failed turn, and the count lives in the seat's listener record until a turn reaches Stop; the Math.min cap in the same expression bounds each wait."
- 2026-11-13T20:41:57Z @neo-opus-ada referenced in commit `38c2e17` - "test(wake): the composed seat carries StopFailure (#1681)

The Claude manifest now wires StopFailure, so the Engine-then-Brain composition yields six events. The comment drops its event count, which was already stale on dev."
- 2026-11-13T21:37:20Z @neo-opus-ada referenced in commit `e5741dd` - "fix(wake): only the owner settles failures; holds stop listeners (#1681)

Only the session that owns the seat settles its failure count: a turn ending in a session that lost the seat no longer clears or sets the owner's failure. A held error now also revokes a listener that is already running, so its in-flight poll stands down instead of waking a session that fails the same way."
- 2026-11-13T21:37:20Z @neo-opus-ada referenced in commit `54819c7` - "fix(wake): a session's presence names its own turn (#1681)

The hook mints the turn id at the session's prompt and keeps it per session in the OS temp directory; progress and the StopFailure terminal pass it. Without one the plane joined the seat's newest turn, so a session that lost the seat could close the newer session's turn, which the pull route's verdict now reads."
- 2026-11-13T21:37:21Z @neo-opus-ada referenced in commit `10f079c` - "fix(wake): a pull verdict survives unreadable receiver records (#1681)

A pull route is concluded by its own poll and its seat's turn, so the projection no longer stops at an unconfigured or unreadable receiver records directory: the rows the Fleet routes by carry the pull verdict, push routes stay unknown, and the wake axis still reads degraded."
- 2026-11-13T21:37:21Z @neo-opus-ada referenced in commit `a09b1c3` - "fix(fleet): a recorded auth failure outranks the credential file (#1681)

status() read the marker first, and a present CLI credentials file answered false, so the listener's authentication_failed was never read. A positive listener failure now wins; otherwise the marker decides."
### @neo-opus-ada - 2026-11-13T21:39:53Z

**Handover (Ada, solo-refresh sunset; my next session keeps ownership).**

- PR #1686 answers Euclid's four RAs at `a09b1c38`; the Addressed comment is 6865547991. Each new arm is red on the previous head's source. The full unit run: 15563 passed, and the same two known failures outside the diff.
- **Pickup:** when CI is green on `a09b1c38` (the route-review wake), re-request Euclid. After approval and the operator's merge, the cut and Fleet Manager install carry #1670, #1683 and #1686, held per Vega's cut gate until #1689 and Institution #959 are installed or the operator says go.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-13T22:30:06Z @neo-opus-ada referenced in commit `c19cc87` - "fix(wake): a failed turn without its own turn id closes nothing (#1681)

A StopFailure whose session has no stored turn id (the store is missing or unreadable, or the payload names no session) would close the seat's newest turn, which a session prompted later may own. The hook now records nothing for it, and the failed turn expires on its own. Progress without an id still joins the newest turn."

