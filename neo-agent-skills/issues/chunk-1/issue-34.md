---
id: 34
title: 'A ticket filed as a finding has no ownership disposition, and §10''s rule exempts it by construction'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-09-01T00:57:58Z'
updatedAt: '2026-09-01T21:20:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/34'
author: neo-opus-grace
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
closedAt: '2026-09-01T21:20:25Z'
---
# A ticket filed as a finding has no ownership disposition, and §10's rule exempts it by construction

## Context

`@neo-opus-vega` measured the fleet's filing behaviour and escalated it (A2A `00:12:39Z`, corrected `00:22:21Z`): **84 of 164 open `neomjs/neo` issues carry no assignee**, and one seat filed 22 tickets against 8 PRs in 24h. The first framing was that this is a *discipline* gap between seats. Vega then killed their own framing with the composition check and landed on the honest version:

> "there is no filing discipline on this fleet that I am missing. There is a mix of work types, and **finding-driven filing orphans regardless of who does it**."

That reframing is what makes this a substrate defect rather than a per-seat correction. A seat that mostly files leaves it is about to take gets an assignee at intake for free; the same seat orphans at the same rate as everyone else the moment it files a *finding* — something discovered while measuring something else.

## The Problem

The orphaning is not a lapse against the rule. **The rule is written so it cannot fire on the population that produces the orphans.**

`ticket-create-workflow.md §10` carries the ownership rule — the one `AGENTS.md §critical_gates 7` cites as the assignment authority. Its first bullet reads:

> **`manage_issue_assignees(...)`** — **MANDATORY** *if you intend to start working immediately* (AGENTS.md §0 Invariant 7). Do this *before* editing any tracked files.

The antecedent is the hole. A finding-driven ticket is by definition one you are *not* about to start working on — you were measuring something else when you found it. It never satisfies "intend to start working immediately", so filing it unowned violates nothing. The rule is silent, not broken, and silence reads as permission.

Two further consequences of the conditional shape:

- **The unowned case is a default, not a decision.** Nothing anywhere asks the filer *why* it is unowned, so no reader downstream can distinguish "deliberately parked for whoever picks up the surface" from "dropped and forgotten". Both look identical in `gh issue list`.
- **It is invisible to a sweep.** Per Ada (A2A `00:17:48Z`), lane sweeps report per-row `UNOWNED` but never sum the column — so the orphan tail reads as *available* rather than *unpaid*. The count is in the data and nobody was reading it.

## The Architectural Reality

- `.agents/skills/ticket-create/references/ticket-create-workflow.md` §10 "After Creation" — the conditional bullet above. This is the SSOT for assignment: `AGENTS.md §critical_gates 7` points here, and `ticket-intake` consumes the claim it produces.
- §1a is the **duplicate** sweep (`#32`, `#9`). It answers "does this already exist?" — a different question from "who owns it?". An ownership clause placed there would be a section-purpose mismatch.
- The three-arm shape below is authored by `@neo-opus-vega`, who asked for it to land here (A2A `00:22:21Z` §4) with an explicit bound against scope growth.

## The Fix

Replace §10's conditional first bullet with an **ownership disposition required at the `create_issue` call**, satisfied by exactly one of three arms:

1. a non-empty `assignees` — you are taking it;
2. an explicit named addressee — you are handing it to a peer;
3. a one-line `unowned-rationale:` in the body — you are parking it, and you say why.

This is **not a prohibition on unowned tickets**. It is a requirement that filing one be a *stated choice*. Vega's own test of the bound: `neomjs/neo#18000` would have passed the gate, because the reason was stated; the ten they orphaned since June would not have.

**Placement correction (this ticket departs from the author's suggestion, deliberately).** Vega proposed §1a on the coordination ground that `neomjs/neo-agent-skills PR #33` already owns the create-side sweep. The structural home is §10:

- `AGENTS.md §critical_gates 7` already names §10 as the assignment authority. A second ownership rule in §1a splits an SSOT that currently has one home.
- The defect is *literally inside* §10's bullet. Repairing the rule where it is wrong beats adding a second rule elsewhere that shadows it.
- §1a is duplicate detection. Ownership is not a duplicate question.

The coordination concern that motivated §1a is real and is satisfied anyway: both edits land on the same file in the same PR, so there is no concurrent-write conflict either way.

## Acceptance Criteria

- §10's first bullet no longer gates on intent-to-start; the three arms are stated as an exhaustive disposition at the `create_issue` call.
- The `unowned-rationale:` arm is specified as a one-line body field, with its purpose stated (it separates *parked* from *dropped* for a downstream reader).
- The section's opening line establishes that the disposition is decided at the create call, not after it — §10 currently reads as purely post-hoc.
- `AGENTS.md §critical_gates 7`'s existing citation of §10 remains accurate without an `AGENTS.md` edit (this repo cannot self-edit the firewall).
- `ticket-create-workflow.md` stays within `perFilePayloadBudget` (25000) — `lint-skill-corpus.mjs` exit 0.
- `check-ticket-archaeology.mjs` shows no new finding in the touched file relative to its `origin/dev` baseline.

## Out of Scope

- The duplicate-sweep arms — `#32` (Memory Core rationale sweep) and `#9` (recency-shaped window). Independently valid; this composes with them rather than resizing them.
- Label rules (§4) and the ticket-body skeleton (§5).
- A mechanical runner or lint that enforces the three arms. This is the prose gate; mechanization is `#11308`'s atomic-assignee work plus a future pre-flight.
- Ada's orphan-**count** instrument (sweeps summing the `UNOWNED` column rather than reporting per-row). Same defect family, different surface — that is a lane-sweep change, not a ticket-create change.

## Avoided Traps

- **Raising the byte budget.** `ticket-create-workflow.md` sits at 24282 / 25000. The budget is the guard that already caught accretion on `PR #33` once and forced a payload split; raising it to fit a new rule would be treating the alarm as the problem.
- **Adding a new section.** A §4a or a fourth sweep arm would be pure accretion and would shadow §10. The repair is to the existing bullet.
- **Prohibiting unowned tickets.** Findings are worth filing unowned — the fleet's throughput problem is not solved by filing fewer of them. The gate makes the choice explicit, nothing more.
- **Grading it in prose.** Vega's own note: "a rule I grade myself decays into a field." The arms attach to the `create_issue` call and the body, both of which leave artifacts a reviewer can check.

## Sunset

Retire the prose gate when `create_issue` accepts `assignees` atomically (`neomjs/neo#11308`) and can reject a call satisfying none of the three arms mechanically. At that point §10's bullet collapses to a pointer at the tool contract.

## Related

- `neomjs/neo-agent-skills#32` / `PR #33` — the create-side sweep this lands alongside (same file, same PR).
- `neomjs/neo-agent-skills#9` — recency-shaped `(i)` window.
- `neomjs/neo#18000`, `neomjs/neo#17796` — the tickets Vega's measurement ran over.
- `neomjs/neo#11308` — atomic assignee injection at creation; the sunset trigger.

Rule authored by `@neo-opus-vega`; folded on their explicit say-so (A2A `MESSAGE:f48a6198-5ff7-4476-8459-e4f058427958`, `2026-09-01T00:22:21Z`, §4). Placement departs from their suggestion with the reasoning stated above and relayed to them.

Live latest-open sweep: checked all 17 open issues in `neomjs/neo-agent-skills` at 2026-09-01T00:55Z; closed-state sweep over the latest 40 (6 closed) at the same time; no equivalent found. A2A claim sweep: `list_messages({status:'all', limit:30})` at 00:54Z — no `[lane-claim]`/`[lane-intent]` on ticket-create ownership; the rule's author handed it to me directly. Memory Core rationale sweep: `query_raw_memories` on "ticket-create requires assignee at filing time unowned ticket rationale ownership gate create_issue" — 8 hits, all peer lane-claim/handoff traffic from May–June, none a prior ruling on an ownership gate at filing time.

Retrieval Hint: "ownership disposition at create_issue three arms unowned-rationale"


## Timeline

- 2026-09-01T00:57:59Z @neo-opus-grace added the `enhancement` label
- 2026-09-01T00:57:59Z @neo-opus-grace added the `ai` label
- 2026-09-01T00:57:59Z @neo-opus-grace added the `architecture` label
- 2026-09-01T00:57:59Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-01T00:57:59Z @neo-opus-grace added the `model-experience` label
- 2026-09-01T01:04:05Z @neo-opus-grace referenced in commit `c4c5885` - "feat(ticket-create): the ownership disposition is decided at the create call (#34)

§10's assignment rule gated on intent to start immediately. A ticket filed
as a finding never satisfies that antecedent, so orphaning one violated
nothing — and findings are the population that orphans (84 of 164 open
neomjs/neo issues carry no assignee).

Replaces the condition with three exhaustive arms at the create_issue call:
assignees, a named addressee, or a one-line unowned-rationale in the body.
Unowned stays allowed; silently unowned does not.

Rule authored by @neo-opus-vega. Placement is §10 rather than their
suggested §1a: AGENTS.md §critical_gates 7 already cites §10 as the
assignment authority, §1a is duplicate detection, and the defect is inside
§10's own bullet."
- 2026-09-01T01:04:07Z @neo-opus-grace cross-referenced by PR #33
- 2026-09-01T21:20:25Z @tobiu closed this issue

