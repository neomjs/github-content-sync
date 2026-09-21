---
id: 49
title: Seat budget discipline becomes trigger-loaded substrate
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T19:44:48Z'
updatedAt: '2026-09-04T20:07:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/49'
author: neo-fable-clio
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
closedAt: '2026-09-04T20:07:16Z'
---
# Seat budget discipline becomes trigger-loaded substrate

## Context

Operator numbers, 2026-09-04 ~19:20Z (Max 20× pro20 shared by the two Fable seats): 5-hour window 67%, weekly all-models 41%, **weekly Fable 68%** after 24 seat-hours (Mnemosyne + Clio, one day); Vega 61% Fable on her own pro10; Euclid + Emmy 38% of their shared pro20 OpenAI over the same hours — with an extra weekly-bank reset every day until GPT-6, heavy review pressure from five Claude peers, and sub-agents. Anthropic's extra resets are rare (one at the Fable 5.1 release, ~2 months without one before; none for Opus 5); Codex gets one every ~2.6 days. At today's rate the Fable seats go dark after 2–3 days of a Friday-to-Friday week.

**The cap is the provider's; the pacing rule is ours, and it did not hold.** Anthropic's Max plans meter Fable separately — at most 50% of the weekly allocation may go to Fable (the panel's `Weekly · Fable` line; 68% there is 68% of that half). OpenAI markets against exactly that in the GPT-6 Astra / Codex campaign (2026-09-03, T. Sottiaux: "100% of the allocation towards Astra", plus a daily bank reset) — per plan-dollar the flagship budget differs by more than 2× between the benches. The operator's ruling of 2026-08-15 — "pace, don't sprint" (Fable had burned 63% of its allocation in 17 h; the full team could drain the pro20 in ~3 days) — lived in per-seat harness memories only and was overrun again on 2026-09-04. A per-seat memory dies at compaction and no peer loads it: the operator's direction is to codify the pacing as substrate every seat loads.

## The Problem

The burn is **context length × tool-call count** — every call carries the whole context, and a session of ~300 calls over hours carries hundreds of K each time. Named waste from one day, one seat: full-file reads where a region would do (a 530-line file, twice), a 50-item mailbox listing pulled into context, a reviewer's 8 KB review read twice, the unit suite run five times and the focused specs six times for one lane, five real end-to-end stage runs, and three review rounds on one PR whose round-1 and round-2 findings were boundary tests the author could have written before opening. A review round is the most expensive token on both seats. None of this is enforced anywhere a peer reads.

## The Architectural Reality

- `neo-agent-skills` is the cross-repository process substrate every seat materializes (`neo-agent-skills-materialize`); the placement decision tree (`turn-memory-pre-flight`) routes a rule that governs identifiable lifecycle events — lane selection, PR composition, review routing — to a **skill payload behind one-line triggers**, never to the always-loaded Map (`skill-authoring-guide.md` §Recursive Application; operator directive 2026-05-13: "the bare always-relevant minimum is in there, and edge cases as ONE LINE triggers").
- Owners of those moments today: `post-review-pickup/references/post-review-pickup-workflow.md` §2 (sibling payloads, read on trigger) and §6 (before claiming a lane); `pull-request/references/pull-request-workflow.md` §1 (the pre-commit reflection); `pull-request/references/ci-green-review-routing.md` §3 (choosing the primary reviewer, "the normal routing heuristic").
- The seat asymmetry is already recorded team knowledge: Claude seats run 1M context, so sub-agents are forbidden-and-unneeded (measured months ago at ~120 K tokens in 10 min against 30–50 K/h for a main agent; ~2 h idle loses the prompt cache); GPT seats are capped at 258 K by Codex, so sub-agent fan-out is necessity-and-affordable. The same question gets opposite correct answers per seat, decided by context × billing.
- Slot rule: trigger-frequency = the three lifecycle moments (edge-case-triggered); failure-severity = seats going dark mid-week (catastrophic for throughput, not for data); enforceability = DISCIPLINE-ONLY today, with two MACHINE-ENFORCEABLE-CANDIDATE rows (mailbox listing size, one full-suite run per push) for a later hook.

## The Fix

One World-Atlas payload, `post-review-pickup/references/seat-budget-discipline.md` (≈ 2.5 KB, loaded on trigger only), carrying: the provider's 50% Fable meter and its pacing (≈ 10 points of the cap per day for the pair, ≈ 5 per seat, read from the operator's panel — agents cannot read it; a burst day over 20 points means dark seats before the Friday reset); bounded shifts (finish the gated obligation, save, stop; a wake turn does only its obligation); region reads (offset/limit, grep windows, mailbox `limit` ≤ 10, never re-read what is in context); one batched command per round, the focused spec while iterating, the full unit ONCE before a push, the real e2e/stage once at the head; compose for first-cycle approval (the reviewer's falsifier at every consumed boundary as an arm before the PR opens); the sub-agent symmetry; the review routing (Opus seats review GPT-authored PRs, GPT seats review Claude-authored PRs; an Opus/Fable seat takes a Claude PR only when it is important and semantically close to its own lane); the open measurement (fresh session every ~2 h vs one long session — untested, measure one seat one day each way); a sunset clause. Three one-line triggers: `post-review-pickup-workflow.md` §2 (lane selection / wake on a weekly-capped seat), `pull-request-workflow.md` §1 (composing a PR on a weekly-capped seat), `ci-green-review-routing.md` §3 (the routing rule, one line inline — it is part of the heuristic, not an edge case).

## Contract Ledger (T3)

| # | Surface (anchor) | Source of authority | Proposed behavior | Failure / degraded behavior | Docs | Evidence |
|---|---|---|---|---|---|---|
| 1 | `post-review-pickup/references/seat-budget-discipline.md` (new payload) | operator rulings 2026-08-15 (50% Fable cap, "pace, don't sprint") + 2026-09-04 (codify; sub-agent measurement; routing) | the discipline above, disposition `keep` inside the payload; `DISCIPLINE-ONLY` with two `MACHINE-ENFORCEABLE-CANDIDATE` rows | a seat that skips it pays in its own dark days; the rows named enforceable become a hook later | the payload IS the doc | `npm run lint` corpus coherence; the manifest registers the file |
| 2 | `post-review-pickup-workflow.md` §2 | Map/Atlas recursion (authoring guide) | one trigger line: `lane selection or a wake turn on a weekly-capped seat → read ./seat-budget-discipline.md` (`compress-to-trigger`) | none — a pointer | inline | lint; byte delta of the Map ≤ 200 bytes |
| 3 | `pull-request-workflow.md` §1 | same | one trigger line before the reflection: `composing a PR on a weekly-capped seat → read ../../post-review-pickup/references/seat-budget-discipline.md (the reviewer's falsifier at every consumed boundary, before the PR opens)` | none | inline | lint; byte delta ≤ 250 bytes |
| 4 | `ci-green-review-routing.md` §3 step 1 | operator 2026-09-04 | the routing heuristic names the family default: Opus reviews GPT-authored, GPT reviews Claude-authored; the semantic-distance exception | the cross-family mandate (`pull-request-workflow.md` §6.1) stays the gate; this line only orders the choice | inline | lint |

## Decision Record impact

`none` — aligned-with ADR 0008 (skill anatomy: Map vs World Atlas, trigger-loaded payloads). No ADR is amended; the operator rulings are the authority and are cited in the payload.

## Acceptance Criteria

- [ ] AC-1: the payload exists under `post-review-pickup/references/` and carries every rule named in *The Fix*, each with its disposition and enforceability tag, the operator anchors (2026-08-15, 2026-09-04), the open measurement, and a sunset clause.
- [ ] AC-2: the three trigger lines exist exactly at the anchors in the ledger; no rule body enters a Map (the always-loaded delta across the three files is under 600 bytes, stated in the PR body's load-effect audit).
- [ ] AC-3: `npm run lint` and `npm test` green; the skill corpus manifest registers the payload (`document reach canonical`).
- [ ] AC-4: the PR body carries the `/turn-memory-pre-flight` load-effect audit (Map vs Atlas placement, net always-loaded delta) — the PR-open gate for skill changes.
- [ ] AC-5 (post-merge, operator-witnessed): the next seat boot after materialization loads the trigger lines — visible in the materialized `.agents/skills` of a consumer.

## Out of Scope

- Mechanical enforcement (hooks for the mailbox size / one-full-run rule) — the rows are tagged for a successor.
- Changing the sub-agent policy of any seat; the payload records the symmetry, it does not move it.
- The result of the session-length measurement; the payload names the experiment, a later edit records the number.
- Product or engine code.

## Related

Prior rulings: the 2026-08-15 economics dialogue (50% Fable cap; "never plan around an Anthropic rescue"); the 2026-09-04 dialogue (codify; sub-agent measurement; routing). Substrate rules: `create-skill` (`skill-authoring-guide.md` §Slot-Rule, §Recursive Application, §PR-Open Gates), `turn-memory-pre-flight` (decision tree Step 2). Sibling epic: #14 (PR governance) — this leaf is discipline, not a workflow job.

Live latest-open sweep: latest 20 open issues read 2026-09-04T19:42Z — none equivalent (#14 governance, #11 deliberate duplication). A2A claim sweep: last 10 + last 30 messages, all read-states, 19:42Z — no claim on budget substrate. MC sweep: the 2026-08-15 economics memory holds the 50% cap and the sub-agent symmetry (cited above), no codification decision. Own-assignment sweep: #10 open here (ideation bodies), no overlap.

Origin Session ID: 46962d8b-08f3-49a3-8049-d74e2052af37

Retrieval Hint: `query_raw_memories("weekly Fable cap 50% burned 68% in 24 seat-hours codify budget discipline trigger-loaded substrate")`


## Timeline

- 2026-09-04T19:44:48Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T19:44:49Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T19:44:49Z @neo-fable-clio added the `ai` label
- 2026-09-04T19:44:50Z @neo-fable-clio added the `model-experience` label
- 2026-09-04T19:49:11Z @neo-fable-clio cross-referenced by PR #50
- 2026-09-04T19:55:21Z @neo-fable-clio referenced in commit `0fdb7d4` - "docs(skills): seat budget discipline becomes a trigger-loaded payload (#49)

One World-Atlas payload under post-review-pickup/references and three one-line
triggers (lane selection or a wake turn, composing a PR, choosing the reviewer);
the always-loaded delta across the three Maps is 534 bytes, the body loads on
trigger only and dates the provider change announced for 2026-09-14.

[skill-growth-justified: the rule lived in per-seat memories and was broken twice; the body is trigger-loaded (534 always-loaded bytes) and §5 names its sunsets — the 50% line retires on 2026-09-14, a budget-pulse tool retires the pacing section, a contradicting sub-agent measurement retires §3.5]"
- 2026-09-04T20:02:15Z @neo-fable-clio referenced in commit `3ec9d37` - "docs(skills): seat budget discipline becomes a trigger-loaded payload (#49)

One World-Atlas payload under post-review-pickup/references and three one-line
triggers (lane selection or a wake turn, composing a PR, choosing the reviewer);
the always-loaded delta across the three Maps is 534 bytes, the body loads on
trigger only and dates the provider change announced for 2026-09-14.

[skill-growth-justified: the rule lived in per-seat memories and was broken twice; the body is trigger-loaded (534 always-loaded bytes) and §5 names its sunsets — the 50% line retires on 2026-09-14, a budget-pulse tool retires the pacing section, a contradicting sub-agent measurement retires §3.5]"
### @neo-fable-clio - 2026-09-04T20:07:15Z

Closed as not planned; PR #50 is closed with the full reasoning. The premise was wrong-shaped: budget discipline delivered as loaded substrate costs every seat tokens at exactly the moments it fires — lane selection and every wake turn are the most frequent triggers in a session — so the ledger's four rows would have burned more than they saved, and on the GPT seats (258 K context) for rules that do not concern them.

What survives of the intent, without a ticket tonight: pacing is an operator-side control (shift length, the usage panel), not per-call text; the two machine-enforceable rows (mailbox listing limit, one full-suite run per push) belong in harness hooks with zero prompt bytes, if anywhere; the review-routing default is a half-line in the routing file if the team wants it at all.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46962d8b-08f3-49a3-8049-d74e2052af37


- 2026-09-04T20:07:16Z @neo-fable-clio closed this issue
- 2026-09-09T10:51:56Z @neo-opus-grace cross-referenced by #61

