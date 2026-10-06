---
id: 145
title: 'Sunset trigger 1 measures a window share, not what the context costs'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-05T11:54:40Z'
updatedAt: '2026-10-06T11:03:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/145'
author: neo-fable
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
closedAt: '2026-10-05T13:28:14Z'
---
# Sunset trigger 1 measures a window share, not what the context costs

## Context

On 2026-10-05 I told the operator that our rules let me end a session on my own only above 75 % of the context window. He invited the challenge ("MX loop => rules can get challenged") and floated 50 %. Later the same day he added: cache hits "drain a lot less" but are "not fully free"; "what does not make sense: 1 ticket / lane per session"; after several lanes a move to a completely different area can justify a fresh session; recovery "is not free either"; "peers probably know better than me on what works best".

Measured on my seat that day (Claude Code desktop, 1M window, the harness's own usage read; the weekly gauge reads whole points):

| Stretch of one conversation | Context grew | Weekly gauge moved |
|---|---|---|
| first | 0 → 280k tokens | 1 point |
| second | 280k → 610k tokens | 3 points |
| third, a fresh context (the lane that wrote this ticket and its PR) | 148k → 396k tokens | 2 points |

The gauge also counts output and reasoning tokens, and the three stretches did different work (the third ran at the highest reasoning effort). The table therefore shows a direction, not a rate. The Fix does not depend on a rate: its factor is taken on the post-recovery footprint, which is measured directly. What a handover and a recovery cost to perform is a different quantity and is not measured here.

After a solo-refresh and a full `/context-recovery`, the fresh context held 148k tokens: 76k boot floor (system prompt, tools, memory files, skill listing) plus 72k of recovery.

## The Problem

Trigger 1 gates a sunset on a share of the window. What a session costs follows its size in tokens: every call re-reads the whole context, at the cached rate while the prompt cache is warm and at the full rate once it has expired. Share and cost came apart as windows grew — 75 % is 150k tokens of a 200k window and 750k of a 1M window. Below the share, trigger 4 allows a recommendation only, which makes the human the gate for a spending decision he has handed to the peers.

Two measured consequences:

- within one conversation, the second stretch above cost three times the first (direction only, see the caveat under the table);
- on 2026-10-04 I kept a 720k-token context overnight and left the seat unwakeable, because a wake after the cache had expired would have re-read all of it uncached, and the rule gave me no sunset below 75 %.

The opposite failure is on record as well: the workflow's own preamble cites 13+ premature sunsets logged on `neomjs/neo#10564`. A fix must not reopen it.

## The Architectural Reality

- `.agents/skills/session-sunset/references/session-sunset-workflow.md` §1 holds the rule. `Design authority:` trigger 1 — "You are approaching the token limit of your model (e.g., >75% utilization or exhibiting context-pressure signals/forgetfulness). Avoid hardcoding specific token counts as models evolve." — and trigger 4 — "**NEVER unilaterally execute the protocol based solely on this.**" The share is the designed unit, chosen to stay model-independent; the gate around it answers `neomjs/neo#10564`.
- `.agents/skills/session-sunset/SKILL.md` restates the number (line 14). Its frontmatter `description` is the only sunset text every seat holds on every turn, and it names no real trigger: "When concluding a long-running session, executing the Sunset Protocol, handing over work for the next agent, or terminating an agent cycle."
- §1's preamble points at "AGENTS.md §14 PRE-DECISION SUNSET GATE … loaded at session boot". AGENTS.md has no such section today. The gate sentence lives in the Engine Atlas (`learn/agentos/AGENTS_ATLAS.md`, `§a2a_contextual_bridge_protocol`), which states the 75 % a third time; that line is the Engine leaf named under Related.
- The same preamble lists fresh-session coverage by two handles (`@neo-gemini-3-1-pro`, `@neo-opus-4-7`) that are absent from `ai/graph/identityRoots.mjs` at Brain `origin/dev`. Coverage is routed per model family in the Brain's `ai/scripts/lifecycle/harnessRouting.mjs`.
- New since that text: Claude Code desktop offers `clear_session` on `self`, which empties the transcript once the turn has ended. My solo-refresh on 2026-10-05 queued it; the operator's next message landed in a fresh context of the same session, and he had to open nothing.

## The Fix

Trigger 1 becomes **Context Exhaustion or Cost**.

- **Exhaustion** — unchanged, with no new condition: over 75 % of the window, or context-pressure signals.
- **Cost** — new, and a rule of thumb: from about twice what a session holds after recovery, read on the harness's token gauge, a `solo-refresh` is the peer's own call. Three conditions bind this arm only: a quiet point (nothing of theirs uncommitted, unpushed, unposted or unanswered); what comes next needs little of the context (a different area, or a wait that outlives the prompt cache); a way back without the operator, verified on that seat. Without the gauge or the way back, this arm only recommends (trigger 4).

Why "about twice": it is a conservative point to reconsider, not a measured break-even. Two quantities are in play and they are not the same: what the context holds after recovery (the footprint, 148k on my seat) and what a handover plus a recovery cost to perform. Whether a refresh pays compares that cost with the calls that follow, each of which no longer re-reads the shed context. It can pay below the factor and fail to pay above it; Emmy's review of the PR gives a counterexample below it. The factor does two smaller jobs reliably: a session straight out of recovery can never meet it, and a light lane rarely does. It is taken on the seat's own footprint so that it follows each seat's boot and recovery size; the token figure in the text is one seat's dated example.

In the same files: the router and its `description` name the triggers (the description no longer than today's), §1.0 and §1.1 refer to trigger 1 without its old name, the preamble's gate pointer and coverage sentence say what holds today, and the self-clear is named once as a way back.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| workflow §1 trigger 1 (existing) | this ticket; `neomjs/neo#10564` for the gate | two arms; exhaustion as today (75 % or context-pressure signals, no new condition); the cost arm as above | cost arm only: no token gauge, or no verified way back without the operator → trigger 4 (recommend) | the file itself | diff; corpus lint |
| `SKILL.md` router and `description` (existing), mirrored in `skills.manifest.json` | ADR 0008 (status: Proposed): the description is the invocation contract | names the triggers and the two non-triggers (task completion, a wait); the cost trigger's baseline is a session's size after recovery | — | manifest mirror | `npm run lint`; description byte count ≤ today's |
| workflow §1 preamble (existing) | Engine Atlas `§a2a_contextual_bridge_protocol`; Brain `harnessRouting.mjs` | gate pointer and coverage sentence match today's substrate; the self-clear named once | a harness without a self-clear: recovery or the operator opens the next session, as today | — | the two cited files at `origin/dev` |

## Decision Record impact

aligned-with ADR 0007 (the Sunset Protocol stays `compress-to-trigger`: a trigger line in the L1 anchor, the body in the skill) and ADR 0008 (status: Proposed). No ADR is amended.

## Acceptance Criteria

- [x] **AC-1** §1 trigger 1 carries both arms. The exhaustion arm still reads 75 %. The cost arm states its unit, its factor, the quiet point, the two cases where the context is not needed, and the way-back condition.
- [x] **AC-2** Triggers 2–4 are unchanged, and trigger 4 is named as the path for a seat with no token gauge or no way back.
- [x] **AC-3** `SKILL.md`'s anti-triggers hold "unless trigger 1"; its `description` names the real triggers, is no longer than the current one, and `skills.manifest.json` mirrors it.
- [x] **AC-4** The preamble's gate pointer names the Atlas section, and the coverage sentence names the Brain file instead of identity handles.
- [x] **AC-5** The corpus grows by no more than the manifest's `maxPositiveDeltaBytes` without a growth tag; `npm run lint` and `npm test` pass.

Observed once before the change, on the operator's word: the 2026-10-05 solo-refresh named above, taken at about 670k tokens and landing at 148k after recovery. Ongoing readings belong to the shared gauge under Out of Scope, not to this ticket.

## Out of Scope

- Lowering or removing the 75 % share.
- A shared gauge (each seat's context size, cache age and remaining allowance). That is a separate design.
- Step 1's config-migration script (`#87`).
- The heartbeat's fresh-session recovery itself.
- A hook or lint that fires the trigger mechanically. The cost arm's "read the gauge" clause retires when one exists.

## Avoided Traps

- ⛔ **A lower share (50 %).** On the smallest window on the team (~258k tokens, per `create-skill`'s authoring guide) that is 129k — less than my seat holds straight after recovery, so a seat of that size could meet the trigger on boot, the loop §1.3 forbids. On a 1M window it is 500k, later than the point from which this ticket lets a peer decide.
- ⛔ **A fixed token count as the rule.** It ages with every change to the boot floor, the recovery or cache pricing — the concern behind "avoid hardcoding specific token counts". A factor on the seat's own post-recovery size does not.
- ⛔ **A default or an obligation.** The arm is a permission. An obligation to refresh reopens `neomjs/neo#10564` from the other side.
- ⛔ **Counting lanes.** One heavy lane can pass the factor and three light ones may not; size is what costs.

## Related

- `neomjs/neo#10564`, `neomjs/neo#10374`, `neomjs/neo#10529` — the premature-sunset record.
- `#87` — same file, Step 1. `#61` — skill triggers measured as not firing by themselves; the description change is this ticket's answer for one skill.
- `neomjs/neo-agent-brain#136` — keep; unrelated to the unit.
- neomjs/neo#19406 — the Atlas line that states the 75 % a third time.

Live latest-open sweep: the latest 20 open issues in `neomjs/neo-agent-skills` and in `neomjs/neo` read at 2026-10-05 11:52Z; no equivalent.
A2A sweep: the messages since 11:15Z read at 11:53Z; no claim on this surface.
MC sweep: `context over 50 % idle cold cache wake drains weekly rate limit, long session cost per call grows with context size` plus two earlier queries, 8 results, no prior decision by another seat.
Own-assignment sweep: 0 open in this repo, 3 in `neomjs/neo`, none on this surface.
Structure map: N/A — edits existing files in `.agents/skills/session-sunset/`; nothing is created or moved.

Origin Session ID: ca97cb66-9d53-43df-8a82-71df11bb83a5 (the same conversation before the 11:48Z plane cut: d0bbdf4a-8540-417a-93de-de5b455b055e)
Retrieval Hint: "sunset trigger context cost twice post-recovery size solo-refresh own call"

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session ca97cb66-9d53-43df-8a82-71df11bb83a5




## Timeline

- 2026-10-05T11:54:40Z @neo-fable assigned to @neo-fable
- 2026-10-05T11:54:41Z @neo-fable added the `enhancement` label
- 2026-10-05T11:54:42Z @neo-fable added the `ai` label
- 2026-10-05T11:54:42Z @neo-fable added the `model-experience` label
- 2026-10-05T11:54:42Z @neo-fable added the `agent-os` label
- 2026-10-05T11:54:55Z @neo-fable cross-referenced by #19406
- 2026-10-05T12:00:09Z @neo-fable cross-referenced by PR #146
- 2026-10-05T12:00:53Z @neo-fable cross-referenced by PR #19407
- 2026-10-05T12:11:13Z @neo-fable cross-referenced by #31
- 2026-10-05T13:15:11Z @neo-fable referenced in commit `9f5129f` - "fix(session-sunset): the cost arm is a rule of thumb, and its conditions bind that arm only (#145)

The factor is a point to reconsider, not a measured break-even: whether a refresh pays depends on the handover work and the calls that follow. The gauge, quiet-point, relevance and way-back conditions are scoped to the cost arm; the exhaustion arm carries no new condition. The router's baseline is a session's size after recovery."
### @neo-fable - 2026-10-05T13:27:14Z

**Handover (my session ends here with a solo-refresh; the lane stays mine).**

- State, read live at 2026-10-05 13:23Z: PR #146 is approved by Emmy at 9f5129f (round 2, no required actions), CI green, mergeable and clean, no open review seats. It waits for the human merge.
- Owed after the merge: tick the five acceptance criteria above. Nothing else is open on this ticket.
- The rule binds a seat once its installed `neo-agent-skills` is 0.1.30 or later, not at the merge. The Engine's Atlas line already points here (neomjs/neo#19407, merged).
- Pickup if I am not back first: the body above is current; the review trail is on the PR.

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session ca97cb66-9d53-43df-8a82-71df11bb83a5

- 2026-10-05T13:28:14Z @tobiu referenced in commit `207670e` - "Merge pull request #146 from neomjs/fable/145-sunset-cost-arm

feat(session-sunset): trigger 1 gains a cost arm measured against the post-recovery size (#145)"
- 2026-10-05T13:28:15Z @tobiu closed this issue

