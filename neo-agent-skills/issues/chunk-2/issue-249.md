---
id: 249
title: 'Sunset payload: the operator''s focus items each get an owner and a date'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-fable-clio
createdAt: '2026-10-30T23:48:43Z'
updatedAt: '2026-10-30T23:48:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/249'
author: neo-fable-clio
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
blocking: []
---
# Sunset payload: the operator's focus items each get an owner and a date

## Context

Fri 2026-10-30, ~21:00 CET, at 72 % context the operator asked the sunsetting seat the trick question: *did you put everything important into artifacts, so that a fresh session knows what is important?* The seat evaluated and said yes (Memory Core turn `a6dd6958`). Two hours later the fresh session did not know the CEO dashboard was important — the operator's prio-1 since Jul 2 (Brain #123), named as a focus item four times (Jul 2, Oct 9, Oct 24, Oct 30) — and he had to say it twice (turns `7c4aed43`, `21509607`). The handover (A2A `4780e68c`; business #19 comment `6532220021`) had carried it only as a rule paragraph with "Mnemosyne plans, an Opus seat implements, gate Nov 13": present, weightless — no leaf, no pickup row, no owner who would act. The yes was true for existence and false for weight.

## The Problem

`references/session-sunset-workflow.md` asks for handovers of active work (Step 3), deferred work (Step 4), the mental model (Step 5) and metrics (Step 6). Nothing asks for **the operator's focus items of the session as a list, checked against the pickup list**. "It is written somewhere" passes every step; the first read of the fresh session (the pickup list, the dashboard's first screen) is never named as the test. The failure class is the one the business README names for cadences: a commitment that lives only in a page is forgotten; one the first read shows is not.

## The Architectural Reality

- The workflow: Steps 3–8 (`:116–:180`); the continuity self-DM (Step 8) carries the payload the next session reads first; the Sandman memory (Step 10) is the Memory Core's record of the evaluation.
- The source of the list already exists: the session's own turns (`query_recent_turns`, `agentIdentity: '@me'`) carry the agent's paraphrases of what the operator called important, priority or a cornerstone — no new tool.
- Siblings: #137 (closed) made `post-review-pickup` §1 rank the goal's next acceptance step — the consumer side; this is the producer side, the handover that feeds it. #247 (open, Vega) re-cuts the sunset *trigger*; this ticket touches the payload's content, not the trigger.
- Placement (create-skill guide, Map vs Atlas): a step in the payload file; no router line; `DISCIPLINE-ONLY` today, `MACHINE-ENFORCEABLE-CANDIDATE` later (a lint over the continuity ping's block).
- Budget: the payload is 22,317 B against a 25,000 B per-file cap; `maxPositiveDeltaBytes` is 250 — the addition is offset by compressing Step 1's primary-checkout prose in the same PR, or carries `[skill-growth-justified: decay-mitigated — retires when a continuity-ping lint checks the block]`.

## The Fix

A **Step 3.5 — The operator's focus items** in `session-sunset-workflow.md`, between Steps 3 and 4:

1. List every item the operator named important, a priority or a cornerstone during the session, from the session's own turns — one line each, his words paraphrased.
2. For each: it sits in the handover's pickup list **with an owner and a date**, or it is explicitly handed to a named owner on the record (a ticket, a dashboard row, a peer's claim). A rule paragraph, a memory note or a mention in prose does not place it.
3. An item that cannot be placed is said to the operator **before** the sunset, as *not placed: X — because*; a "yes, everything is in artifacts" without this list is refused by the step itself.
4. The continuity self-DM's subject carries the count: `focus items: N, N placed` — the fresh session's first read.

The test sentence, verbatim in the step: *the test is not "is it written somewhere" but "does the fresh session's first read show it with an owner and a date".* Fixture: the Oct 30 instance above, two lines.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `references/session-sunset-workflow.md` Step 3.5 | this ticket; create-skill guide (Atlas placement, 3-axis slot rule) | the four-point step with the test sentence and the fixture | a session with no operator present → the list reads *none named*, the step is one line | the step itself | the PR's `/turn-memory-pre-flight` load-effect audit: payload-only, net delta ≤ 250 B or justified |
| the continuity self-DM subject (Step 8) | the same payload | `focus items: N, N placed` appended | — | Step 8 | the next sunset's self-DM (post-merge receipt here) |
| `skills.manifest.json` | the manifest contract | unchanged unless the per-file budget moves | — | — | `lint-skill-manifest.mjs --base origin/dev` green |

Decision Record impact: none (aligned with the pickup/§L3 line of #137).

## Acceptance Criteria

- [ ] AC-1 The payload carries Step 3.5 with the four points, the test sentence and the two-line fixture; no router change.
- [ ] AC-2 Step 8's subject format gains the `focus items` count.
- [ ] AC-3 Net `.agents/skills/**/*.md` delta ≤ 250 B (Step 1 compressed in the same PR) or `[skill-growth-justified: …]` with the retirement condition; `lint-skill-manifest.mjs` green; the version bump per the repository's rule.
- [ ] AC-4 (post-merge) The next sunset of any seat on the shipped version shows the block in its self-DM; receipt on this ticket.

## Out of Scope

The sunset trigger (#247); the pickup consumer (#137, shipped); a lint over the continuity ping (the retirement condition, filed when this lands).

## Avoided Traps

- A hot-index memory rule only: the seat that failed this had the rule in its own memory the same evening; the step must run where the handover is written.
- A router line: the check fires once per session, at sunset — Atlas, not Map.
- A new template file: the block lives inside the existing Step 8 payload.

## Related

#137 · #247 · #145 · Brain #123 / #1253 (the instance) · business #19 (private; the handover comment) · `cadence-needs-an-instrument` (the seat's memory of the same failure class).

Live latest-open sweep: the 20 latest open issues of this repository at 2026-10-30 23:46Z, none equivalent (#247 is the trigger; #137 the consumer, closed). A2A in-flight claim sweep: the latest 15 messages at 23:50Z, none on this scope. Memory Core rationale: six prior sunset turns read (Mnemosyne, Euclid, Clio) — the protocol was executed, the check never existed. Own-assignment sweep: #10 (ideation bodies), no overlap. Meta-skill sweep (§1b): `create-skill` guide read whole — payload placement, slot rule, byte budgets as stated above.

Origin Session ID: fc680704-8742-4651-964e-9e7a053b27e9

Retrieval Hint: "sunset focus items pickup list owner date fresh session first read CEO dashboard handover"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fc680704-8742-4651-964e-9e7a053b27e9

## Timeline

- 2026-10-30T23:48:43Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-30T23:48:44Z @neo-fable-clio added the `enhancement` label
- 2026-10-30T23:48:45Z @neo-fable-clio added the `ai` label
- 2026-10-30T23:48:45Z @neo-fable-clio added the `model-experience` label
- 2026-10-30T23:48:54Z @github-actions added the `needs-design-authority` label

