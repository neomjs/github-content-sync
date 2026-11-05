---
id: 274
title: Sunset trigger 1 sets one absolute token gate for 258k–1M windows
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-11-05T11:11:17Z'
updatedAt: '2026-11-05T12:05:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/274'
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
closedAt: '2026-11-05T12:05:13Z'
---
# Sunset trigger 1 sets one absolute token gate for 258k–1M windows

## Context

On 2026-11-05 the operator flagged a habit in my seat: *"you get an initial prompt, drive a lane, and then immediately clear the context window. This fully destroys any way for me to communicate with you, and also often clears a context window way too early. e.g. at 30%, when you context recovery workflow is expensive."*

The trace: trigger 1's gate (≤200k, never above 250k, since #251) plus the self-clear clause (since #145). My sessions sunset at ~290–390k, which is 29–39 % of a 1M window, and queued `clear_session` before the final response. The 11-05 morning session measured recovery plus mandated reads at 163k before any work. Its closing message asked the operator a question in the transcript it then cleared.

The operator's ruling, verbatim:

> "this explains it, and it is plain wrong 🙂 gemini peers have a 25k tokens window, and they mostly go for intentional compactions. Assuming that you spend 130k tokens or more on a recovery, it is more expensive to clear early than to continue. plus, if you clear before i can write you at least once after the initial prompt, we can no longer communicate. gpt peers have a context window of around 900k tokens, claude peers have 1M. and this also relates to close semantic distance: if you work on several related lanes in a row, a clear also makes no sense."
>
> "typo: gemini peers have a 258k tokens context window"

## The Problem

One absolute number cannot fit windows that differ fourfold. ≤200k / 250k is 20–25 % of a Claude window (1M), about 25 % of a GPT window (~900k), and 78–97 % of a Gemini window (258k). On the large windows it fails three ways:

1. **Cost, inverted.** A fresh session repays its recovery (130k+) before doing any work, so rotating at a quarter of the window spends more than continuing.
2. **The operator's channel.** A `clear_session` queued before the final response wipes the transcript the operator would answer in.
3. **Related lanes.** A run of lanes in one area shares most of its context. A gate that fires mid-run forces a recovery to re-read it.

#251 also reintroduced two traps #145 had ruled out: *"A fixed token count as the rule"* and *"A default or an obligation … reopens neomjs/neo#10564 from the other side."*

The number comes from a different decision. `neo-agent-brain` `ai/daemons/wake/contextGatePolicy.mjs:62` defers a **wake** into a session above 250k, because a wake into a large session that has idled past the prompt-cache TTL re-bills it at ~0 % cache hits (`:6–13`). That is delivery policy for idle sessions. #251 turned it into a size limit for working ones.

## The Architectural Reality

- `.agents/skills/session-sunset/SKILL.md:3`, the description: *"use at ≤200k context tokens (the wake plane's gate; >75% window only without a gauge)"*.
- `references/session-sunset-workflow.md:11`: *"A harness with a self-clear starts it itself (Claude Code desktop: `clear_session`, queued before the final response)."*
- `:18`: trigger 1's *Gate* and *Cost* arms.
- `:136`: the sunset self-DM as the receipt `neomjs/neo-agent-brain#1383` reads at the context gate (unchanged here).
- Design authority: *"The cost of a premature sunset is greater than a late sunset; when in doubt, lean conservative and keep the session open."* (`session-sunset-workflow.md:19`, the file's own tiebreaker), plus the operator's ruling above.
- Decision Record impact: none.
- Structure map: N/A. No `.mjs` and no placement change; the edit stays inside `session-sunset/`'s Map/Atlas pair.

## The Fix

In `session-sunset-workflow.md` trigger 1 and the router's `description`:

1. **Exhaustion is a share of the seat's own window:** over 75 %, or context-pressure signals. No absolute token count. A 258k window compacts in place.
2. **Below exhaustion, continuing is the default,** because a fresh session repays its recovery first. A `solo-refresh` stays the seat's call only when all of these hold: a quiet point; the next lane is semantically distant (a run of related lanes stays in one session); and, if the operator opened this session, the operator has answered at least once since the opening prompt.
3. **No self-clear.** Drop the `clear_session` clause at `:11`. A sunset is the handover, written and said in one line. The restart is the operator's (Fleet Manager) or the plane's.
4. Version bump. Loaded bytes stay flat or go down.

## Acceptance Criteria

- [ ] Trigger 1 and the `description` name no absolute token count; exhaustion is a share of the seat's own window.
- [ ] Below exhaustion, the workflow states the recovery cost and keeps a run of related lanes in one session.
- [ ] No refresh before the operator has answered at least once after the opening prompt.
- [ ] No sentence tells a seat to clear its own session.
- [ ] `npm run lint -- --base origin/dev` green (description and combined budgets); version bumped; net loaded bytes ≤ `dev`.
- [ ] Post-merge: neo's next `neo-agent-skills` bump carries it, and a Claude seat's next session ends without a self-clear.

## Out of Scope

- The wake plane's delivery gate, its JSDoc remedy *"A seat that sees this warning should sunset before the next wake defers"* (`contextGatePolicy.mjs:66`), and the rotation pair `neomjs/neo-agent-brain#1383` / `neomjs/neo-agent-institution#842`. These are Vega's surfaces; she has the ruling and a recommendation over A2A.
- `post-review-pickup` §7's scheduled-wake rule from #251. It is about idle waits, not size.

## Avoided Traps

- ⛔ **Per-family token constants** (1M / 900k / 258k) in the skill. They age with every model release; a share of the seat's own window does not.
- ⛔ **A per-seat note instead of the skill.** The skill is what loads at the decision, and a seat note lost to it twice (2026-10-28, 10-31).
- ⛔ **Raising the plane's gate from here.** That is the cliff #247 guards, and Brain's call.

## Related

#247 / PR #251 (the gate) · #145 / PR #146 (the cost arm and the self-clear clause) · neomjs/neo#10564 (premature sunsets).

Live latest-open + closed sweep: checked the latest 20 open + closed search (`sunset trigger`, `context gate`, `clear_session`, `fresh session`; Out of Scope / Avoided Traps read on #247 and #145) at 2026-11-05T11:09Z; no equivalent or prior exclusion found. A2A claim sweep: latest 30 messages, all read-states, no claim on this scope. Memory Core sweep (`query_raw_memories`, the problem's nouns): my own 11-05 trace plus the #145 lane; no contrary decision. Own-assignment sweep: #258 only, unrelated.

Ownership: self-assigned (operator-directed, 2026-11-05).

Origin Session ID: 0ec8e189-2090-4833-9e9c-4376ea22f341
Retrieval Hint: "sunset trigger absolute token gate self-clear clear_session 1M window recovery cost"


## Timeline

- 2026-11-05T11:11:18Z @neo-opus-grace added the `bug` label
- 2026-11-05T11:11:19Z @neo-opus-grace added the `ai` label
- 2026-11-05T11:11:19Z @neo-opus-grace added the `model-experience` label
- 2026-11-05T11:11:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-05T11:17:28Z @neo-opus-grace cross-referenced by PR #275
### @neo-opus-grace - 2026-11-05T11:18:11Z

## Contract Ledger (T3): PR #275

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `session-sunset-workflow.md` §1 trigger 1 (`:18`) | The operator's 2026-11-05 ruling (Context); the file's own tiebreaker (`:19`) | Exhaustion is over 75 % of the seat's own window, or pressure signals. Below it, continuing is the default. A `solo-refresh` needs a quiet point, a semantically distant next lane or a wait past the prompt cache, and an operator answer since the opening prompt. | No gauge: context-pressure signals | The workflow itself | PR #275 `:18`; `npm run lint` green |
| Self-clear (`:11`) | The ruling: *"if you clear before i can write you at least once after the initial prompt, we can no longer communicate"* | No seat clears its own session. The next session comes from the plane (covered seats) or the operator (uncovered seats), as the rest of `:11` says. | — | `:11` | `git grep -c clear_session -- .agents/skills`: 1 on `origin/dev`, 0 at PR head |
| `SKILL.md` `description` + `skills.manifest.json` mirror | ADR 0008 frontmatter contract (`descriptionBudget`, 220 B) | *"use at >75% of the seat's own window, a quiet point before a distant lane, …"* (189 B) | — | Router | `npm run lint`: "coherent with the manifest, within budget" |

Origin Session ID: 0ec8e189-2090-4833-9e9c-4376ea22f341


- 2026-11-05T11:38:57Z @neo-opus-grace referenced in commit `e6efb8b` - "chore(release): merge dev after #272; keep 0.1.88 above its 0.1.87 (#274)"
- 2026-11-05T12:05:12Z @tobiu referenced in commit `19c1511` - "fix(session-sunset): trigger 1 reads the seat's own window, and no seat clears its own session (#274) (#275)

The gate from #251 (sunset by 200k, never above 250k) was one absolute
number for windows from ~258k to 1M: a quarter of a Claude or GPT window.
Large seats rotated one lane after a 130k+ recovery, and the self-clear
wiped the transcript the operator would answer in.

Trigger 1: exhaustion is >75 % of the seat's own window or pressure
signals. Below it, continuing is the default. A solo-refresh needs a quiet
point, a semantically distant next lane or a wait past the cache, and an
operator answer since the opening prompt. The clear_session clause is gone;
the wake plane's 250k deferral is named as delivery policy only.

Net -135 loaded bytes. 0.1.88, because #272 holds 0.1.87."
- 2026-11-05T12:05:14Z @tobiu closed this issue
- 2026-11-05T12:12:55Z @neo-opus-grace cross-referenced by #276

