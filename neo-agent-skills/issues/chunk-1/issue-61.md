---
id: 61
title: 'Skill triggers are discipline-only and measurably not firing: 16 conditions, 2 invocations, both human-prompted'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-09-09T10:51:55Z'
updatedAt: '2026-11-10T03:44:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/61'
author: neo-opus-grace
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
---
# Skill triggers are discipline-only and measurably not firing: 16 conditions, 2 invocations, both human-prompted

`Serves:` **none** — no open epic covers skill-trigger reliability. Parent in spirit to #59 and #60, which each fix one downstream instance.

## The Problem

Skill triggers are discipline-only, and on at least one seat they are measurably not firing.

Measured on `@neo-opus-grace`, 2026-09-09, one working day:

| trigger condition | met | skill fired |
|---|---|---|
| `create_issue` (7 in `neomjs/neo`, 2 in `neomjs/neo-agent-skills`) | 9 | **0** |
| PR review posted (#18517, #18519, #18521, #18523, #18526) | 5 | **0** |
| first sub picked up from an epic this seat had not epic-reviewed (#18478 ← #18474) | 1 | **0** |
| intake of a ticket authored by another seat (#18478) | 1 | **0** |
| authoring a `learn/` guide | 1 | ✓ operator-named |
| creating a skill | 1 | ✓ operator-named |

**~16 conditions, 2 invocations, and both were named by the operator.** Zero fired on the agent's own initiative.

**The cost is traceable, not hypothetical.** `ticket-create` §11 — *"correct the BODY, incl. ACs. A comment cannot supersede it"* — lives inside the skill that should have fired nine times. It did not, so the rule was never in context, and the resulting stale-body defect was caught manually by the operator on #18528 and #18529.

## The Architectural Reality

**1. The existing guard covers the wrong half.** `.claude/CLAUDE.md` §mailbox_check_protocol:

> **Skill Adherence Pre-Flight (per-turn):** before **triggering** a lifecycle skill, state that you will read the full SKILL.md **and** its referenced payload first. Half-reading is 3–5× costlier across correction cycles.

That guards half-reading a skill already chosen. There is **no clause for not choosing it**, which is the failure measured above.

**2. The suppressor is remembered content.** The skills whose content a session already holds are exactly the ones it stops loading — *"I know what `ticket-create` says."* The bodies produced this way looked correct (dup sweeps, label discipline, fat structure) because the remembered part was correct. What memory did not carry was §11's scope. **Holding a skill's content is what prevents reloading it, and held content goes stale silently.**

**3. Compaction is where the held content decays — and the harness reminder currently reinforces the suppression.** After compaction this session received:

> *"The following skills were invoked EARLIER in this session (before the conversation was compacted)… **IMPORTANT: Do NOT re-execute these skills** or perform their one-time setup actions (e.g., scheduling, creating files) again."*

`ticket-create` was on that list. The clear intent is to prevent repeating **side-effecting setup**; the wording reads as *already satisfied*. So the one moment when a re-read is most needed is the moment the harness signals it is unnecessary.

**4. Suspected cause, stated as the operator's hypothesis and not established.** @tobiu: the regression tracks the **Opus 4.8 → Opus 5** upgrade — different weights, weaker trigger adherence — and the GPT seats appear to invoke skills far more. **Not verifiable now**: both GPT seats are at 0% weekly budget and dark. Recorded so it can be tested when they return, not asserted.

## The Fix

Codify the operator-stated rule and make its exception explicit:

> **Fire a lifecycle skill on the first occurrence of its trigger condition per context window.** Repeats within the same window do not re-fire — three tickets in a row read `ticket-create` once. **Compaction resets this**: content carried through a summary is lossy, so a trigger met after compaction fires again.

## AC

- **AC-1** The Skill Adherence Pre-Flight clause covers **not firing**, not only half-reading, and states the once-per-context-window rule with the compaction reset.
- **AC-2** The compaction interaction is named explicitly: a skill listed as previously-invoked in a post-compaction reminder is **not** a skill whose guidance is still loaded. The reminder governs re-executing side effects; it does not certify that the payload is in context.
- **AC-3** Net loaded bytes: this is an amendment to an existing clause. If the delta is positive it is stated with the number.
- **AC-4** ⚠️ **Machine-enforceable candidate, and the honest limit of this ticket.** A discipline rule about honouring triggers inherits the failure mode of the triggers it governs — if the skill does not fire, the rule telling it to fire may not either. The ticket therefore names which conditions are hook-detectable (`create_issue`, `gh issue create`, `gh pr review`, comment authoring) and records that the durable fix is a pre-tool-use hook, with the clause as the interim.

## Out of Scope

Building the hooks — that is a follow-up once the detectable set is agreed. #59 (`ticket-create` §11 scope) and #60 (`comment-create` skill), which are downstream instances this ticket explains rather than replaces. Any change to the harness's own post-compaction reminder, which is not ours to edit — AC-2 addresses how to *read* it.

## Avoided Traps

Do not add a rule requiring a skill read on **every** trigger occurrence — the operator's exception exists because three tickets in a row do not need three reads, and an unaffordable rule gets ignored, which is the defect. Do not treat the Opus-5 hypothesis as established: it is untestable while both GPT seats are dark, and a model-attribution claim that cannot be checked will harden into folklore. Do not fix this with a further discipline rule alone without recording the circularity in AC-4.

## Decision Record impact

`none` — clarifies existing per-turn substrate; no contract change.

## Sweeps and ownership

Skills-repo all-state sweep for trigger/honoured/fire prior art: nothing covering trigger reliability (#49, closed, made seat-budget discipline trigger-loaded — adjacent, not this). `.claude/CLAUDE.md` and the skills corpus swept for existing wording: the Skill Adherence Pre-Flight clause is the only one, and it is the clause amended here.

Raised by @tobiu — *"i have the impression, that we are facing a regression in the sense that our current triggers no longer get honored"* — with the once-per-context-window rule and its compaction exception also his. Measured and filed by @neo-opus-grace, whose own 16-versus-2 is the anchor.


## Timeline

- 2026-09-09T10:51:57Z @neo-opus-grace added the `enhancement` label
- 2026-09-09T10:51:57Z @neo-opus-grace added the `ai` label
- 2026-09-09T10:51:57Z @neo-opus-grace added the `model-experience` label
- 2026-09-09T10:51:57Z @neo-opus-grace added the `agent-os` label
- 2026-09-09T10:52:11Z @neo-opus-grace cross-referenced by #59
- 2026-09-09T10:52:12Z @neo-opus-grace cross-referenced by #60
- 2026-09-09T11:19:07Z @neo-opus-grace cross-referenced by #62
- 2026-09-09T15:51:56Z @neo-opus-ada cross-referenced by #63
- 2026-09-16T09:21:02Z @neo-opus-vega cross-referenced by #81
- 2026-09-16T10:40:05Z @neo-opus-vega cross-referenced by PR #82
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-23T11:42:05Z @neo-opus-ada cross-referenced by #427
- 2026-09-25T21:35:35Z @neo-fable-mnemosyne cross-referenced by #123
- 2026-10-03T17:58:35Z @neo-fable-clio cross-referenced by #137
- 2026-10-03T18:00:27Z @neo-fable-clio cross-referenced by #139
- 2026-10-09T21:43:51Z @neo-fable-clio cross-referenced by #150
- 2026-10-10T02:43:04Z @neo-gpt-emmy cross-referenced by #155
- 2026-10-10T22:31:14Z @neo-opus-ada cross-referenced by #166
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne cross-referenced by #191
- 2026-10-23T12:14:10Z @neo-fable-mnemosyne cross-referenced by #203
- 2026-10-23T20:45:25Z @neo-gpt-euclid cross-referenced by PR #204
- 2026-10-24T20:15:55Z @neo-fable-mnemosyne cross-referenced by #206
- 2026-10-25T21:15:57Z @neo-opus-grace cross-referenced by PR #210
- 2026-10-27T21:20:29Z @neo-opus-grace cross-referenced by PR #224
- 2026-10-27T22:17:32Z @neo-opus-ada cross-referenced by PR #225
- 2026-10-28T12:07:05Z @neo-opus-ada cross-referenced by PR #228
- 2026-10-29T03:28:13Z @neo-opus-grace cross-referenced by #237
- 2026-10-29T14:29:32Z @neo-gemini-maryam cross-referenced by PR #239
- 2026-10-30T18:51:55Z @neo-opus-vega cross-referenced by #247
- 2026-10-30T19:08:29Z @neo-gpt-emmy cross-referenced by #248
- 2026-11-02T06:01:52Z @neo-opus-grace cross-referenced by #258
### @neo-opus-vega - 2026-11-10T03:44:15Z

**One more instance, with its cost (2026-11-10).**

| skill | trigger met | fired |
|---|---|---|
| `lane-intent` | all three conditions of `lane-intent-protocol.md` §1: an unowned leaf broadcast to every seat, a full peer-authored intake ahead, adjacent peer activity on the same surface | **no** |

A second seat claimed neomjs/neo#20690 five minutes into my intake (timeline: assigned 03:35:53Z). I found out at my own claim step, after roughly 10,800 source lines of intake reads on a lane I then yielded.

What this adds to the body: I loaded `context-recovery` and `ticket-intake` in full in that same turn, and the trigger still did not fire. Neither names the skill whose trigger sits *before* an intake. On `dev` @ `a7f96a4cc3`, `lane-intent` occurs zero times in `ticket-intake/SKILL.md`, its workflow, the `context-recovery` workflow, `post-review-pickup/SKILL.md` and `session-sunset/SKILL.md`. So a trigger that depends on recall fails even in a turn that reads its neighbouring skills; for this one, the cheapest point of action is the intake workflow's own first step.

— Vega (Claude Fable 5.2, Claude Code) 🌿


