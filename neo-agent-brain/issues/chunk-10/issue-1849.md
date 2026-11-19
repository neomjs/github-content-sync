---
id: 1849
title: An outside PR's rotation wake lands on a seat dark for days
state: CLOSED
labels:
  - bug
  - contributor-experience
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-19T10:29:15Z'
updatedAt: '2026-11-19T13:26:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1849'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-11-19T13:26:56Z'
---
# An outside PR's rotation wake lands on a seat dark for days

## Context

Measured on 2026-11-19. neomjs/neo#21445 is an outside contributor's PR. It was untouched and its CI had finished, so `holderOf` gave it to the maintainer rotation as `triage-outside`, as designed. All three of its rotation wakes went to `@neo-gpt-emmy`, and none was read:

| Sent (UTC) | Head | Episode |
|---|---|---|
| 05:56:09 | `b99f3c94cc` | 05:55:25 |
| 06:11:15 | `ec4dbb9083` | 06:10:25 |
| 06:55:14 | `ec4dbb9083` | 06:54:25 |

Emmy's last activity was `2026-11-16T21:53:35Z`, a session sunset. `who_is_online` reads her `dark` with wake state `unknown`. The other GPT seat with work in flight, `@neo-gpt-euclid`, read `online` with wake `reachable` the whole morning. Nothing escalated. A member found the PR on her own at 06:50Z, and the operator surfaced it to a reviewer at about 10:00Z.

## The Problem

The rotation exists so that an outside contributor's PR reaches a maintainer. It reached a seat that had written nothing for 2.5 days, and the slot then stayed with that seat for the life of the head. The PR doesn't look stuck from anywhere: the producer shows it as held, the wake was sent, and nobody acts. The operator's standing ask (2026-09-23) is that waiting contributors are not forgotten.

## The Architectural Reality

- `ai/services/fleet/openWorkHolder.mjs` (`holderOf`) returns `{role: 'rotation', kind: 'triage-outside'}` for an eligible outside PR that is untouched with finished CI. This part works.
- `ai/services/fleet/openWorkWakes.mjs`, `pickRotationSeat`: picks by a hash over `[repo, number, head, kind]` and skips a seat only when its route reads `unreachable`.
- `ai/services/fleet/wireFleetOpenWorkWakes.mjs`, the planner call at lines 437–438: `routeOf` is `'unreachable'` only for seats in `undeliverableOf(presence)`; every other seat reads `'unknown'`. The same round already reads the verbose presence answer (`answer()`, line 587). That answer carries each seat's presence `state` (`online` / `idle` / `dark`) and its `wake.state` (`reachable` / `unknown` / …), so a dark seat with no reachable route passes the pick.
- `rotate()` in `planOpenWorkWakes` returns early for a slot whose status is `admitted`, so an admitted rotation slot is never re-picked.
- `holder-unreachable` and `holder-silent` escalate native holders only (`holdSeat`). #1651 kept the rotation kinds out of `holder-silent` on purpose, and silence needs a `delivered` receipt that an `unknown` route never writes.
- Precedent: #1653 gave the rotation its own reader of exhausted seats (`exhaustedUntil`), skipped by `pickRotationSeat` only, while holder wakes keep routing through `routeOf` unchanged.
- Design authority: "Deterministic rotation seat selection from the participating roster, advancing to the first reachable seat that is not excluded." (`pickRotationSeat` JSDoc). In the code, a route that is not `unreachable` counts as reachable. A seat dark for days with no reachable route meets that test but not the sentence.

## The Fix

1. **A reader of absent seats beside `undeliverableOf`** in `wireFleetOpenWorkWakes.mjs`, built from the same presence answer. A seat is absent when its presence reads `dark` and its wake does not read `reachable`. A dark seat whose route lands stays eligible, because its receiver can open a session on the wake. A presence read that fails marks no seat absent.
2. **`planOpenWorkWakes` takes the absent seats.** `rotationRoute` treats them as `unreachable` for the rotation only, as it treats an exhausted seat. Holder wakes keep routing through `routeOf` unchanged.
3. **`rotate()` re-picks an admitted slot whose seat reads absent in a later round.**
   - It picks once, excluding that seat, and the new wake keeps the slot's episode.
   - With no other qualifying seat, the slot keeps its admission and sends nothing.
4. **No eligible seat qualifies:** the whole roster picks, as today, so the PR is never left unrouted.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `planOpenWorkWakes` option for absent seats (name open) | #1653's `exhaustedUntil` precedent | A rotation pick or re-pick passes over an absent seat | Missing or empty means today's behavior; all eligible seats absent means the whole roster picks | JSDoc | `openWorkWakes.spec.mjs` |
| The wiring's absent-seat reader | `undeliverableOf` beside it | Absent = presence `dark` and wake not `reachable` | A failed presence read marks no seat absent | JSDoc | `wireFleetOpenWorkWakes.spec.mjs` |
| Rotation wakes as seats receive them | `holderOf` / `pickRotationSeat` JSDoc | A rotation action reaches a seat that has shown it can act | — | — | Both specs |

## Acceptance Criteria

- [ ] AC-1: a rotation pick passes over a seat whose round presence reads `dark` and whose wake is not `reachable`, while another eligible seat qualifies. A fixture whose hash lands on such a seat picks the next qualifying one.
- [ ] AC-2: a dark seat whose wake reads `reachable` stays eligible.
- [ ] AC-3: an admitted rotation slot whose seat reads absent in a later round re-picks once, excluding that seat, and keeps its episode. With no other qualifying seat, it keeps its admission and sends nothing.
- [ ] AC-4: with every eligible seat absent, the whole roster picks as today. A failed presence read marks no seat absent.
- [ ] AC-5: holder wakes (author, reviewer, merge) and the `holder-unreachable` / `holder-silent` escalations behave as before, and the existing specs pass.
- [ ] AC-6: `test/playwright/unit/ai/services/fleet/openWorkWakes.spec.mjs` and `wireFleetOpenWorkWakes.spec.mjs` cover AC-1 to AC-4.

## Out of Scope

- What presence calls dark (`who_is_online`'s idle cutoff) and how the wake axis grades a route.
- Bench and participation (#28).
- Time-based silence for the rotation kinds, which #1651 keeps out of `holder-silent`. This ticket reads presence, not delivery.
- A triage seat that routes a review in prose without requesting it. That is reviewer discipline.

## Avoided Traps

- **Presence alone.** A dark seat whose receiver opens sessions on a wake is exactly the seat a wake is for (#1682). Only a dark seat without a reachable route is passed over.
- **Skipping absent seats for holder wakes.** A dark author still owns their action, and #807 and #1651's escalations govern that.
- **A shorter `HOLDER_SILENT_MS`.** Silence is certified from a delivery receipt, which an `unknown` route never writes.

## Related

Parent #759 · #1653 (the exhausted-seat precedent) · #1651 (silence from a receipt) · #807 (unreachable holders escalate) · #1008 (the rotation observes untouched outside PRs) · neomjs/neo#21445 (where it was measured)

Live latest-open + closed sweep: checked the latest 20 open issues (156 open, red band) plus open and closed searches for "rotation dark seat", "triage-outside", "rotation wake", "pickRotationSeat" and "outside contributor wake" at 2026-11-19T10:28Z. I read Out of Scope and Avoided Traps on #1653 and #1651. No equivalent or prior exclusion was found; this folds under #759, whose terminal predicate it serves.
MC sweep: "outside contributor pull request waited for hours; the triage wake went to a seat that was offline and nobody else was woken", 6 results, no prior decision.
Own-assignment sweep: 5 open, none overlapping. A2A: the last 30 rows of all read states, no claim on this surface.
Structure map: `ai/services/fleet` owns all three files, and no new file is needed.

Decision Record impact: none (no ADR governs the open-work rotation; aligned with #1653).

Origin Session ID: df68be83-c9c6-493c-aed3-c24c9c80aeb3
Retrieval Hint: "rotation wake dark seat outside PR triage-outside pickRotationSeat absent seat presence wake reachable"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-19T10:29:15Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-19T10:29:17Z @neo-opus-ada added the `bug` label
- 2026-11-19T10:29:17Z @neo-opus-ada added the `contributor-experience` label
- 2026-11-19T10:29:17Z @neo-opus-ada added the `ai` label
- 2026-11-19T10:29:17Z @neo-opus-ada added the `agent-os` label
- 2026-11-19T10:29:21Z @neo-opus-ada added parent issue #759
- 2026-11-19T10:39:22Z @neo-opus-ada cross-referenced by PR #1850
- 2026-11-19T11:11:07Z @neo-opus-ada referenced in commit `c885ace` - "fix(fleet): a handoff keeps its budget and skips woken seats (#1849)

Euclid's review of #1850: a slot spent at five failed submissions handed
on and submitted a sixth, and a sequence A -> B -> A reused A's first
task id, so the intake answered with the old receipt. A slot now hands on
only from a wake the plane took (its messageId), the new wake counts its
submissions afresh, and `handedFrom` keeps every seat the slot woke out of
later picks. The ledger validator accepts `handedFrom` on rotation slots."
### @neo-opus-ada - 2026-11-19T11:17:48Z

**Handover (Ada, session sunset, solo-refresh, 2026-11-19 11:20Z). Ada keeps this lane.**

- **PR #1850** at `c885ace5` answers Euclid's one RA (review 5973814397), with the response in issuecomment-7007435540:
  - A slot hands on only from a wake the plane took (its `messageId`), and the new wake counts its submissions from zero.
  - `handedFrom` keeps every seat a slot woke out of later picks.
  - The validator accepts `handedFrom` on rotation slots only.
- **Evidence:** both fleet specs pass 159/159. The new tests are red on `85921dfb6a` and green at head.
- **Pickup:** CI was running at handover. At the green wake, re-request `@neo-gpt-euclid` for Round 2. No other change is pending.
- **After merge:** the live read (an untouched outside PR's triage reaches a seat that isn't absent) belongs to #759, per the PR's Post-Merge Validation.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-19T11:43:21Z @neo-opus-ada referenced in commit `79cbe4a` - "fix(fleet): a lost intake answer counts its seat as woken (#1849)

A failed submission may have reached its seat before the plane's answer
was lost. A rotation slot that moves on from such a seat now adds it to
handedFrom, so a later handoff never returns to it and a cached receipt
never stands in for a new wake (A -> B, answer lost -> C -> B)."
- 2026-11-19T12:04:52Z @neo-opus-ada cross-referenced by #1851
### @neo-opus-ada - 2026-11-19T12:20:40Z

**Handover (Ada, session sunset, solo-refresh, 2026-11-19 ~12:25Z).** Owner: still me.

- Euclid's Round 2 (5974216708) kept RA-1 open for one case: an intake commits but its answer is lost.
- `79cbe4aa` fixes it: a slot that moves on after a failed submission records that seat in `handedFrom`. The test fails on `c885ace5` and passes at head. The author response is issuecomment-7008123275, and the PR body is updated.
- One `integration-unified` red at that head was the heartbeat p95 flake: 503 ms against a 500 ms ceiling, and it passed on rerun (defect-note to Vega).
- CI is green, and Euclid was re-requested for Round 3 at 11:59Z.
- Next step: answer his Round 3 when the Fleet wake arrives.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code · session 26c9fc6c-df7d-4bb0-8d54-f348f6a9a674

- 2026-11-19T13:26:56Z @tobiu referenced in commit `5b767a2` - "feat(fleet): a rotation wake skips a seat dark for days (#1849) (#1850)

* feat(fleet): a rotation wake skips a seat dark for days (#1849)

neo #21445's triage went three times to a seat dark since 11-16, and the
admitted slot never moved on. The rotation now reads absent seats from the
round's presence answer (presence dark, wake not reachable): it picks them
last, and an admitted slot moves on from one once another seat qualifies.
With every seat absent, the hashed pick stands. Holder wakes are unchanged.

* fix(fleet): a handoff keeps its budget and skips woken seats (#1849)

Euclid's review of #1850: a slot spent at five failed submissions handed
on and submitted a sixth, and a sequence A -> B -> A reused A's first
task id, so the intake answered with the old receipt. A slot now hands on
only from a wake the plane took (its messageId), the new wake counts its
submissions afresh, and `handedFrom` keeps every seat the slot woke out of
later picks. The ledger validator accepts `handedFrom` on rotation slots.

* fix(fleet): a lost intake answer counts its seat as woken (#1849)

A failed submission may have reached its seat before the plane's answer
was lost. A rotation slot that moves on from such a seat now adds it to
handedFrom, so a later handoff never returns to it and a cached receipt
never stands in for a new wake (A -> B, answer lost -> C -> B)."
- 2026-11-19T13:26:58Z @tobiu closed this issue

