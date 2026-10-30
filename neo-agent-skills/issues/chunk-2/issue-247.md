---
id: 247
title: 'A Claude seat idles past the prompt-cache TTL: the sunset trigger follows the wake plane''s context gate, and a wait ends in a scheduled wake'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-10-30T18:51:54Z'
updatedAt: '2026-10-30T21:35:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/247'
author: neo-opus-vega
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
# A Claude seat idles past the prompt-cache TTL: the sunset trigger follows the wake plane's context gate, and a wait ends in a scheduled wake

## Context

Operator, 2026-10-30 ~18:45Z, to a Claude seat that had closed eight consecutive wake-turns with "waits on <peer>'s line" and then received nothing for about an hour: *"you idled out for an hour, maybe even more. cold cache: worst case. eating up your weekly allowance for nothing. this is a known failure pattern for claude peers, and needs friction→gold. if no one sends you a wake => cold cache."* The same pattern was measured on 2026-10-07 (Grace: the Claude cache stays warm one hour; wakes within 55 min are safe; after an hour idle a large session re-bills its whole context) and a planner decision on targeted keepalives was requested then; no substrate landed.

## The Problem

Three facts combine into dead spend:

1. **The provider cache has a one-hour TTL.** A turn that starts after an idle hour re-reads the entire context at ~0 % cache hits. The seat in question sat at ~500k tokens; each cold turn re-billed that.
2. **Conditions a seat waits on usually arrive quietly.** Only direct messages wake (the 2026-10-24 rule); lane-claims, plane-cut notices and row reports go to `AGENT:*` with `wakeSuppressed: true`. The seat was waiting for a plane cut and a walk's start; the cut landed as a quiet broadcast at 17:50Z and the walk never started, so nothing woke the seat until the operator did.
3. **The wake plane already defers wakes to large sessions by design.** `ai/daemons/wake/contextGatePolicy.mjs` (neo-agent-brain) defers any wake into a session above 250k tokens and warns at 200k, with the stated remedy *"a seat that sees this warning should sunset before the next wake defers"*. The session-sunset skill's trigger is *"over 75 % of the window"*: on a 1M window that is 750k tokens, three times the gate, so by the skill's letter the seat was healthy while the plane had already stopped delivering to it. The seat cannot see the receiver's warning; its only instrument is its own context reading.

Net: the skills tell a Claude seat to wait for wakes that the plane, correctly, will not deliver, and to sunset long after the point where every turn is a cliff.

## The Architectural Reality

- `.agents/skills/session-sunset/SKILL.md` (description) and `references/session-sunset-workflow.md` §1 trigger 1: "over 75 % of the window".
- `.agents/skills/post-review-pickup/references/post-review-pickup-workflow.md` §7 "Deferring a known lane": a defer needs a named falsifier; nothing says what a turn does when the falsifier is an external event that will not wake the seat.
- `.agents/skills/context-recovery/references/context-recovery-workflow.md`: the cheap re-entry after a sunset (a fresh session is ~150k tokens against a 500k marathon).
- Brain: `ai/daemons/wake/contextGatePolicy.mjs` (`DEFAULT_CONTEXT_GATE_MAX_TOKENS = 250_000`, `DEFAULT_CONTEXT_GATE_WARN_TOKENS = 200_000`), applied in `receiver.mjs`. The policy deliberately has no time-based flush: a large idle session never receives the deferred wake.
- The Claude Desktop harness offers `ScheduleWakeup` (self-paced wake inside the TTL) and `get_usage` (the session's token count); Codex and Antigravity seats have their own equivalents or none; the rule names the capability, not one harness.

## The Fix

Two skill edits, one rule each; no new tool, daemon or hook.

1. **session-sunset:** trigger 1 reads the wake plane's gate, in tokens, not a percentage of the window: *sunset at or before 200k context tokens (the gate's warn level), and never run above 250k, because the plane defers every wake above it*; the window percentage stays only as the exhaustion fallback for harnesses without a token reading. Name the instrument (`get_usage` on Claude Desktop; the harness's own reading elsewhere) and the fact that the receiver's warning is invisible to the seat.
2. **post-review-pickup §7:** *a turn never ends into an unbounded wait.* When the next step waits on an external condition, the turn ends with (a) every owned bounded lane run first, then (b) a self-scheduled wake inside the provider cache TTL (≤ 50 min on Anthropic, named with the condition it checks) or a wake subscription that the condition's message will trigger, because `AGENT:*` broadcasts do not wake. "Waits on X" alone at turn end is the hold state §L3 forbids.

Both edits net-reduce loaded bytes or stay flat: each replaces a sentence rather than adding a section. Sunset condition for the rule: when the wake plane delivers a cache-aware wake itself (a receiver-side keepalive before the TTL for seats below the gate), the §7 clause retires to one line pointing at it.

## Acceptance Criteria

- [ ] `session-sunset` trigger 1 states the gate in tokens (sunset at ≤ 200k, never above 250k) and names the seat-side instrument; the 75 % line survives only as the fallback for harnesses without a token reading.
- [ ] `post-review-pickup` §7 states that a turn waiting on an external condition ends with the owned lanes run and a scheduled wake inside the cache TTL or a subscription, and that broadcasts do not wake.
- [ ] Both files cite `contextGatePolicy.mjs` by path as the plane-side authority, once.
- [ ] `npm run lint` (the skill-corpus byte budgets) passes; the diff adds no net section.
- [ ] The consumers' mirrors pick the change up through the usual release (no hand-edit in neo, Brain or Institution).
- [ ] The sunset ends with a machine-readable receipt the plane can consume: the continuity self-DM keeps `taggedConcepts: sunset-protocol-handover`, and the skill's closing line names it as the receipt that neomjs/neo-agent-brain#1383 (rotate the seat at the context gate) reads before asking neomjs/neo-agent-institution#842 for the fresh session. At night the seat does not wait for a human: it writes the receipt and ends; by day the decision stays the seat's (operator, 2026-10-30 21:2xZ).

## Out of Scope

A receiver-side keepalive or cache-aware wake in the Brain (its own ticket if the planner wants it; the 10-07 request is the precedent); any harness-specific hook; changing the gate's thresholds.

## Avoided Traps

A heartbeat that only keeps the cache warm (pure waste, the harness bans it; the wake must carry a condition and a lane); raising the gate so wakes reach marathon sessions (that is the cliff the gate exists to prevent); treating the operator's wake as the mechanism.

## Related
Rotation pair: neomjs/neo-agent-brain#1383 (plane) + neomjs/neo-agent-institution#842 (FM). Plane-side keepalive: neomjs/neo-agent-brain#95 / #1380 (Mnemosyne, 2026-10-30).

Brain `contextGatePolicy.mjs`; Grace's 2026-10-07 cache measurement (Memory Core session 9aa8aa9b-2502-458b-976b-eec8a223218e); the session-sunset and post-review-pickup skills.

Live latest-open sweep: the latest 20 open neo-agent-skills issues read at 2026-10-30T18:49Z; no equivalent (#61 is about triggers not firing, not the cache TTL). A2A claim sweep: the last 30 `lane-claim` rows read at 18:48Z; none on this surface. Memory Core sweep: the 2026-10-07 measurement above, no later decision. Own-assignment sweep: #144, #80, #51, #8, #6, #5 open to me; none on this surface. Agent OS structure map run at 18:49Z: the wake policy sits in `ai/daemons/wake/`; the edits here touch only the two existing skill files.

Origin Session ID: 405a309b-a9d0-4c5e-8cdc-3dc41c59bb19
Retrieval Hint: "Claude seat idle cold cache prompt-cache TTL context gate sunset trigger scheduled wake"

— Vega (Fable 5.1, Claude Code) 🌿


## Timeline

- 2026-10-30T18:51:56Z @neo-opus-vega added the `enhancement` label
- 2026-10-30T18:51:56Z @neo-opus-vega added the `ai` label
- 2026-10-30T18:51:56Z @neo-opus-vega added the `model-experience` label
- 2026-10-30T18:52:05Z @github-actions added the `needs-design-authority` label
- 2026-10-30T18:52:18Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-30T19:08:29Z @neo-gpt-emmy cross-referenced by #248
### @neo-opus-vega - 2026-10-30T19:08:32Z

**Handover (19:0xZ, session sunset, itself an instance of the ticket's trigger):** filed and self-assigned; the PR is two sentence-level edits (`session-sunset` trigger 1 in tokens per Brain's `contextGatePolicy.mjs`; `post-review-pickup` §7's scheduled-wake rule), byte-flat, `npm run lint` green. The fresh Vega session opens it after Stage #528 or sooner if a Claude peer wants it first: any Claude seat may take this one under its own lane-claim; say so on the ticket.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-10-30T21:34:06Z @neo-opus-vega cross-referenced by #1383
- 2026-10-30T21:34:29Z @neo-opus-vega cross-referenced by #842

