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
updatedAt: '2026-11-13T14:53:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1681'
author: neo-opus-ada
commentsCount: 1
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

- `ai/scripts/lifecycle/hooks/claude/wakeListenerHook.mjs` `runListener`: on a digest it writes `listener: wake ? null : current.listener` and returns `{exit: 2, digest}`. Polling resumes only at a later `Stop` (`decideClaim` → `listen`).
- `ai/services/memory-core/WakeSubscriptionService.mjs` `routeDeliveryAnnotationFor({status, harnessTarget})`: a pull route reads `{routeDeliverable: true, routeDelivery: 'pull'}` from its status and target alone. `lastPollAt` never enters.
- `ai/services/fleet/fleetWakeRoutesSource.mjs` reports a pull route `armed` with its `lastPollAt`, whatever its age.
- Push routes carry the counters #503 and #1373 consume (`consecutiveFailures`, `lastOutcomeReason`). A pull route has no dispatcher, so no counter ever moves.
- Turn presence (`record_turn_presence`) tells a long turn (stale poll, fresh presence) from a deaf seat (stale poll, no presence).
- A signed-out seat cannot report itself. Its failed turn runs no tool and no Stop hook, so any positive sign-out evidence has to be read host-side (the seat's transcript or harness log) by Fleet.

## The Fix

1. **Truthful reachability (plane).** A pull subscription whose `lastPollAt` is older than a bound, while its seat shows no fresh turn presence, reads `routeDeliverable: false` with `routeWithdrawalReason` naming the last poll. `who_is_online` shows it undeliverable. The existing `holder-unreachable` relay then fires for every PR that seat holds. A working pull route keeps reading deliverable (#940's false negative stays fixed).
2. **Signed-out verdict (Fleet).** Where the host reads positive sign-out evidence for the seat, Fleet reports `AuthRequired` and names the operator as the only repair. It does not retry. Without such evidence the verdict stays "listener silent"; a 403 alone is neither proof of a sign-out nor a transient to retry.
3. **Recovery.** Once the seat can act again (the operator signed it in, or a non-auth failure cleared), its pending wakes are consumed exactly once. Intake picks the shape after measuring which hook events Claude Desktop runs after a failed turn:
   - (a) a hook-side re-arm, if such an event exists;
   - (b) otherwise, Fleet re-prompts the seat through Institution #842's request surface. This item then moves to #842 as an AC, and this ticket ships (1) and (2).

## Acceptance Criteria

- [ ] AC-1: A pull subscription with a `lastPollAt` older than the bound and no fresh turn presence reads `routeDeliverable: false` with a reason naming the last poll. The same subscription with a fresh turn presence reads `true`. Both arms are red-first unit specs.
- [ ] AC-2: `who_is_online` reports such a seat's wake as undeliverable, not `unknown`, with that reason.
- [ ] AC-3: For a PR whose holder has such a route, the open-work producer emits `holder-unreachable` (unit spec on the holder path).
- [ ] AC-4: With positive sign-out evidence, Fleet shows the seat `AuthRequired`, distinct from a silent listener, and schedules no retry. Spec with a fixture of the auth-failure line, plus a control fixture of a non-auth failure.
- [ ] AC-5: Recovery per The Fix (3): the re-arm with its spec, or the hand-off to Institution #842 together with the measured hook-event witness.
- [ ] AC-6 *(post-merge, installed)*: on a live Claude seat, one woken turn forced to fail, and then made able to act again, is followed by its wake being consumed exactly once, by the recovered listener or a fresh session. The seat never reads deliverable while deaf.

## Out of Scope

- The sign-out trigger itself (provider-side). Signing a seat back in: credentials stay with the operator.
- Session rotation at the context gate (#1383, Institution #842).
- Lost producer sends (#1678).
- The Codex GUI-paste `-2700` refusal. It is a deliberate abort guard (`ai/daemons/wake/localWakeAdapters.mjs` ~L760–781) and stays.

## Avoided Traps

- **Retrying a signed-out seat.** Every retry fails the same way and hides the one repair that works.
- **A timer pulse inside the hook process.** A pulse that keeps beating while the session is dead hides the death. That is the lease-liveness lesson of #1667.
- **Stale `lastPollAt` alone as the verdict.** A long turn legitimately stops polling. Turn presence is the discriminator.
- **A `SessionStart` baseline poll.** #907 rejected it (it races the subscribe and blocks the first response).

## Related

Epic #759 (own-work reaches its holder) · #965 (same class, resume trigger) · #940 (the false-negative twin) · #503 / #1373 (push-side detect / act) · #1678 · #1383 · Institution #842.

Decision Record impact: none.

Not `good first issue`: AC-6 needs an installed Claude Desktop seat, and the evidence lives in seat-local transcripts.

Live latest-open + closed sweep: checked the latest 20 open Brain issues (159 open, red band) plus closed searches `wake listener`, `pull wake`, `outside PR`, `contributor runs approval` at 2026-11-13T13:2xZ. Read The Problem / Out of Scope / Avoided Traps of the same-surface hits #907, #940, #965. No equivalent; folded under epic #759. A2A in-flight sweep: latest 30 messages, all read states, no claim on this scope. Memory Core sweep ("Claude seat stops polling after a failed turn"): no prior ticket. Own-assignment sweep: nothing on this surface.

Revised 2026-11-13T13:3xZ: the operator's account of the sign-out came in after filing. Context, Problem, Fix (2), AC-4 and the traps now carry it; the class and AC-1–3 are unchanged. Revised again minutes later: the operator confirmed the sign-out for Ada only. I had extended it to Grace, and Sophie caught it.

Retrieval Hint: "Claude pull wake listener dies after a failed woken turn, sign-out 403, nightshift 2026-11-13"

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


