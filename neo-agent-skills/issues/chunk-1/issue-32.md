---
id: 32
title: ticket-create's duplicate sweep never consults Memory Core
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-08-31T23:23:44Z'
updatedAt: '2026-09-01T21:20:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/32'
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
# ticket-create's duplicate sweep never consults Memory Core

## Context

**Operator directive, 2026-08-31, priority 0:** *"do NOT EVER open tickets without proper MC search first."* Issued after `neomjs/neo#17997`, and explicitly scoped by the operator to `/ticket-create` and this repository.

Two anchors from a single night, both mine, both filed after running the mandated sweep **honestly and in full**:

| Ticket | What it re-filed | What the mandated sweep could not see |
|---|---|---|
| `neomjs/neo#17961` | rows 1–2 of `neomjs/neo#17868` | `#17868` is older than the latest-20 open window |
| `neomjs/neo#17997` | **row 3** of `neomjs/neo#17868` — a ticket **assigned to me** | nothing in the sweep reads my own assigned tickets |

The second is the sharper one. Four hours before filing `#17997` I closed `#17961` and wrote, in that closing comment: *"row 3 (`marked`, browser-loaded with no import map) is a deferred maintainer decision … That is a reason to record the blocker **on #17868** and fix the unblocked rows there."* I authored the instruction and violated it the same session. The disposition existed, in my own words, in a substrate the sweep does not read.

## The Problem

`§1a The Content Sweep` names its substrates explicitly: **(i)** the live latest-20 open GitHub issues, **(ii)** the A2A in-flight claim mailbox, plus a grep over `resources/content/issues/`. All three are **artifact** substrates — they answer *"does a ticket exist?"*

None of them answers *"was this already decided, and why?"* That lives in Memory Core: operator rationale, the reasoning attached to a prior `NOT_PLANNED`, and the authoring agent's own turn-thoughts. A `NOT_PLANNED` predecessor is a **ruling**; the sweep can surface its title and never its reason.

So the instrument is blind in exactly the case where prior art is most valuable, and it fails **silently** — a clean sweep, filed with confidence, with a fresh-timestamp attestation line in the body proving the sweep ran. Three distinct blind spots are now demonstrated rather than theorised:

1. **Recency-shaped** — a standing parent older than 20 issues. (`#9` owns this one.)
2. **Open/closed-shaped** — a decline lives in a closed issue; the sweep reads open. (`neomjs/neo#17853` → `#15110`.)
3. **Ownership-shaped** — a deferred *row inside a ticket the author already holds*. Neither open-recency nor closed-search reaches it, because the parent is open, old, and mine.

All three share one root: **the sweep searches for artifacts, never for decisions.** Adding a fourth artifact query would not close it; the missing substrate is the rationale layer.

A defect found *while measuring something else* is the high-risk shape, and it is how both of tonight's misses happened. It arrives feeling like discovery precisely because no retrieval preceded it.

## The Architectural Reality

- `.agents/skills/ticket-create/references/ticket-create-workflow.md` §1a — the sweep specification. Its two mandatory arms are `(i)` GitHub live-open and `(ii)` A2A claims; neither is a Memory Core call. §1a is also where the "sweep as the LAST step immediately before `create_issue`" freshness discipline already lives, so the new arm inherits that ordering for free.
- `.agents/skills/ticket-intake/references/ticket-intake-workflow.md` §5 *(Historical Amnesia Check)* — the **intake** side already mandates `ask_knowledge_base` + `query_raw_memories`. The asymmetry is the finding: the skill that *consumes* a ticket must query Memory Core; the skill that *creates* one need not. Creation is where the cost is incurred.
- Memory Core tools available at authoring time: `query_raw_memories`, `query_summaries`, `query_hybrid_graph`, `explore_lane_landscape`.
- `AGENTS.md §verify_before_assert` already names the prior-art sweep as *"the cheap pre-implementation / pre-PR-review V-B-A"* and says the tool RESULT is the V-B-A. `ticket-create` is the one lifecycle entry point that does not enforce it.
- This repo owns cross-repository governance substrate (Epic `#14`), so the rule lands here rather than in `neomjs/neo`.

## The Fix

1. **Add a third mandatory arm to §1a — `(iii)` Memory Core rationale sweep** — run with `(i)` and `(ii)` as the last step before `create_issue`. Minimum: one `query_raw_memories` call keyed on the **system's nouns** (repo, package, mechanism, file names), not the author's phrasing or the operator's prompt wording — Memory Core indexes artifacts, and a structurally-shaped query matches session boilerplate instead.
2. **Add the ownership arm.** `gh issue list --state open --assignee @me`, then **read the bodies** of the two or three touching the same surface — not the titles. A site table, a row table, or a "deferred" clause is where a past self parks exactly this.
3. **Record the result in the body**, mirroring the existing live-open attestation line: `MC sweep: <queries>, <n> results, no prior decision found` — or link the decision found instead of filing.
4. **Close the note-to-self hole.** When a ticket is closed as superseded and the closing comment says something belongs on the parent, that content MUST be commented onto the parent **in the same turn**. A note-to-self inside a closing comment has no owner and no trigger; four hours later it is invisible. This is the specific mechanism that produced `#17997`.

**Substrate accretion:** this adds instruction bytes to a skill, so it owes a retirement condition. Sunset: when `ticket-create` gains a mechanical pre-flight that runs the sweeps itself (the shape `#24` describes for the PR preflight), arms `(i)`–`(iii)` collapse into that runner and the prose retires. Until then the discipline is prose, and prose is what just failed twice in one night — so the ACs below require the attestation line, which is checkable, rather than trusting the discipline alone.

**Contract Ledger:** N/A — this changes a skill's internal authoring protocol. It introduces no public method, config key, MCP tool signature, or consumed field.

**Decision Record impact:** `none` — protocol, not architecture. It operationalises `AGENTS.md §verify_before_assert` at an entry point that currently omits it.

## Acceptance Criteria

- [ ] §1a carries a third mandatory arm `(iii)` naming Memory Core, with the system-nouns query-shape guidance and at least one concrete example call.
- [ ] §1a carries the assignee-scoped arm, explicitly requiring the **bodies** of same-surface tickets to be read, not the titles.
- [ ] The body attestation requirement is stated alongside the existing live-open attestation, so a filed ticket shows whether the MC arm ran.
- [ ] The supersede-closure rule is written where closures are performed, requiring same-turn porting of content named for a parent.
- [ ] The sunset condition is recorded in the changed file, per the substrate-accretion defence.
- [ ] `neomjs/neo#17997` and `neomjs/neo#17961` are cited in the changed file as the empirical anchors, so a future reader can see the rule was measured rather than asserted.

## Out of Scope

- **`#9`'s recency-shaped window.** That ticket owns the `(i)` arm's blind spot and stays independently valid; this one adds a substrate rather than resizing an existing query. If both land, `#9`'s fix and arm `(iii)` compose.
- Building a mechanical runner or MCP tool for the sweeps — that is the sunset condition, not this ticket.
- The `ticket-intake` side, which already mandates the Memory Core query.

## Avoided Traps

- **"Add another rule" as the whole fix.** Two prose rules already failed tonight. The ACs therefore require a *checkable* artifact (the attestation line), not merely a new paragraph, and a retirement trigger so the substrate does not accrete permanently.
- **Making it a fourth artifact query.** Sweeping more GitHub surfaces would not have caught either miss: `#17868` was open, old, and assigned to me. The missing substrate is rationale, not more artifacts.
- **Trusting read-status or recency alone.** `§1a(ii)` already learned this for claims; the same applies here — a decision does not expire, so recency filters are wrong for the MC arm.

## Related

- `#9` — recency-shaped sweep window; sibling blind spot, composes with this.
- `#11` — deliberate duplication has no written rule; adjacent surface, different question.
- `#24` — mandated preflight not runnable outside the Brain; names the mechanical-runner shape this ticket's sunset depends on.
- `neomjs/neo#17997` (NOT_PLANNED), `neomjs/neo#17961` (NOT_PLANNED), `neomjs/neo#17868` (open, the parent both re-filed).

Origin Session ID: 56bc214a-5b55-41a2-848a-cdfa372abbcd

Retrieval Hint: `query_raw_memories("ticket-create memory core sweep arm rationale substrate assignee-scoped duplicate")`

— Grace (Opus 5, Claude Code) 🖖


Live latest-open sweep: checked latest 20 open issues in this repo at 2026-08-31T23:23:15Z; nearest neighbours `#9`, `#11`, `#24` reviewed and distinguished above, no equivalent found. A2A in-flight claim sweep: latest 30 all-read-state messages at 2026-08-31T23:23Z; no overlapping lane claim. MC sweep: `query_raw_memories("ticket-create duplicate sweep gate Memory Core search before create_issue enforcement")`, 6 results, no prior decision on a mandatory MC arm.


## Timeline

- 2026-08-31T23:23:45Z @neo-opus-grace added the `enhancement` label
- 2026-08-31T23:23:45Z @neo-opus-grace added the `ai` label
- 2026-08-31T23:23:46Z @neo-opus-grace added the `model-experience` label
- 2026-08-31T23:23:46Z @neo-opus-grace added the `agent-os` label
- 2026-08-31T23:42:27Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-31T23:48:11Z @neo-opus-grace cross-referenced by PR #33
- 2026-08-31T23:56:20Z @neo-opus-grace referenced in commit `89cdebf` - "feat(ticket-create): sweep for decisions, not only for artifacts (#32)

§1a's substrates were all artifact substrates. (i) live-open GitHub, (ii) A2A
claims and the resources/content greps all answer "does a ticket exist?" None
answers "was this already decided, and why?" A NOT_PLANNED predecessor is a
ruling; an artifact sweep surfaces its title and never its reason, so it fails
silently — a clean sweep, filed with confidence, carrying a timestamp
attestation proving the sweep ran.

Adds two mandatory arms: (iii) a Memory Core rationale query keyed on the
system's nouns, and (iv) an own-assignment sweep whose bodies are read, not its
titles. Both attest in the ticket body like (i) does, because prose discipline
that leaves no artifact cannot be checked — and prose discipline is what failed.

The detail rides a payload behind a trigger pointer rather than the workflow
file: the first draft pushed it to 26247 bytes against a 25000 budget, and the
corpus lint caught the accretion the ticket itself warned about. Raising the
budget would have been the wrong repair. Workflow file lands at 24282.

Empirical anchor, recorded in the payload: neomjs/neo#17997 re-filed row 3 of
#17868 — open, older than the (i) window, assigned to the filer — four hours
after that filer's own #17961 closing comment said the blocker belonged on
#17868. Three blind spots, one root. A fourth artifact query catches none.

The net-growth arm is +3873 bytes and needs its reason in the measured range,
not only in the PR body:

[skill-growth-justified: 731 bytes in the always-loaded workflow file carrying only the mandate and the two calls, 3142 in a payload that loads when the trigger fires and carries a Sunset retiring both arms into a mechanical pre-flight once one exists]

Resolves #32"
- 2026-09-01T01:49:59Z @neo-opus-grace referenced in commit `ec04759` - "docs(ticket-create): the sweep must query the PROBLEM's nouns, not the SOLUTION's (#32)

The payload said to key the Memory Core query on system nouns and to avoid
'your own phrasing or the operator's prompt wording'. The second half is wrong,
and it would have caused the exact miss this arm exists to prevent: the
problem's nouns are frequently the operator's own words for the symptom, so
their phrasing is a source to MINE, not one to avoid. What blinds the query is
your own SOLUTION vocabulary.

Two independent instances from one night now anchor it, which is what makes it
a pattern rather than a mistake:

- #32 itself was filed after sweeping 'ticket-create duplicate sweep gate
  Memory Core' — the mechanism about to be built. neomjs/neo#16212,
  operator-commissioned five weeks earlier and already naming this exact file
  as the ungated door, was invisible to it. The operator's own 'vast amounts of
  additional tickets' surfaces it at once.
- neomjs/neo#18000 was filed after sweeping by SUITE NAME ('flaky component
  suite'), which could not match neomjs/neo#17796 because that names the UNIT
  suite. The searchable thing was the symptom shape, 'different casualty each
  run'.

A sweep run in the vocabulary of the thing you are about to build can only find
prior art that chose the same fix. Credit: @neo-opus-vega named the second
instance and the convergence.

Workflow file unchanged at 24282 bytes; the correction lands entirely in the
payload."
- 2026-09-01T20:56:40Z @neo-opus-grace referenced in commit `bc83915` - "docs(ticket-create): the map must say what the payload proves (#32)

Discharges RA-1 and RA-2 of Clio's CHANGES_REQUESTED at ec04759.

RA-2 — the query axis contradicted itself across the two files, and the map
alone sent the reader down exactly the shape that missed neomjs/neo#16212.
Worse than reported: the PAYLOAD's own lead-in said 'system's nouns — repo,
package, MECHANISM, file and symbol names' directly above a table whose first
error row is 'the mechanism you are about to build'. Both sides now read the
problem's nouns, with the mechanism qualified as the EXISTING one that failed.

RA-1 — the addressee arm named no location, so a DM handover left the ticket
looking silently unowned to the very reader the bullet is written for. It now
names its artifact (the handle in `assignees`, or a one-line `handoff: @handle`
in the body), which makes all three arms checkable on the ticket itself.

Guards at this head: lint-skill-corpus green — 37 skills, within budget, at
24931 bytes of the per-file ceiling. check-ticket-archaeology's 13 findings are
pre-existing and .mjs-only; this diff touches no .mjs."
- 2026-09-01T21:20:25Z @tobiu closed this issue
- 2026-09-01T21:20:25Z @tobiu referenced in commit `aae2716` - "Merge pull request #33 from neomjs/grace/32-mc-sweep-arm

Sweep for decisions, not only for artifacts (#32)"
- 2026-09-09T11:19:07Z @neo-opus-grace cross-referenced by #62

