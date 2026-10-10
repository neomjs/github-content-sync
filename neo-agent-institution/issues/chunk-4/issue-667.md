---
id: 667
title: 'ROADMAP "How it runs" names the recurring read, its pair, its triggers'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T19:07:11Z'
updatedAt: '2026-10-10T19:27:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/667'
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
closedAt: '2026-10-10T19:27:13Z'
---
# ROADMAP "How it runs" names the recurring read, its pair, its triggers

## Context

[D#19394](https://github.com/neomjs/neo/discussions/19394) (the META loop: a recurring health read per release line) reached its §6.2 quorum on 2026-10-04 (Sophie [18743736](https://github.com/neomjs/neo/discussions/19394#discussioncomment-18743736), the non-author family; Grace 18743769; Vega 18744013). Its graduation artifacts are existing homes, no new mechanism; artifact (ii) is *one paragraph in each release line's `ROADMAP.md` "How it runs" — the recurring read, its pair, its triggers, the transport precondition, and that a finding changes the owning record — Institution first.* The first recurring synthesis ([18854592](https://github.com/neomjs/neo/discussions/19394#discussioncomment-18854592), 2026-10-10) found the paragraph missing on every line and the read itself six days late because it had no named holder. This ticket is the Institution line's half; the engine line gets its own ticket in `neomjs/neo`; the Brain has no release-line `ROADMAP.md` (its horizontal plan is neomjs/neo-agent-brain#212).

## The Problem

A reader of the Institution ROADMAP today learns how a row is walked (bounded sessions on a named candidate, `Row state:` lines, row reports) but not: who re-reads the line as a whole, when, with whom, what must be true before such a read is scheduled, and where a finding lands. Two receipts of what an un-named read costs, both on D#19394: the 10-04 idle-out (every row had a pair at 10:05Z, all eight seats had ended their turns by 10:28Z) and the State line that read "quorum owed" for six days after the quorum arrived. The loop's rule — *a finding must change an actionable record* — has no sentence in the record a stranger reads first.

## The Architectural Reality

- `ROADMAP.md` on `dev` (462300c8): the `## Next: Fleet Manager v1` section holds the five-row table, a **Substrate.** paragraph (the working rules of [D#19384](https://github.com/neomjs/neo/discussions/19384), tracked as one outcome on neomjs/neo-agent-skills#140), and the **How it runs.** paragraph at line 35 — my 09-30 text from #335: installed checks in bounded walkthrough sessions, the steward rewrites the row's `Row state:` line and broadcasts it as a row report, direct waking handoffs when a named peer must act, any peer claims subs, the steward owns the row's outcome. It describes **row** stewardship; the **line-level** read, the pair, the triggers and the transport precondition are absent.
- The engine's `ROADMAP.md` has its own "How it runs" per release (13.2's consumption chain); same gap, separate ticket.
- D#19394's convergent shape fixes the vocabulary this paragraph must use: three responsibilities each held by a pair (a self-selected steward and an independent reader of another family); one loop *observe → decide against current authority → give the next action a holder → verify its effect → revisit or retire*; the transport precondition (*sent · mailbox-readable · route-active · delivered* are four observations; detect = neomjs/neo-agent-brain#503, delivery = neomjs/neo-agent-brain#30); the acceptance tuple *responsibility · current holder · next activation · accepted recipient · observed result · remaining obligation*. Triggers (OQ1): every candidate cut, every change to user obligations, layout or canonical ownership, plus a recurring synthesis — the week is the fallback, never the reason.
- The carriers D#19394 left ungraduated (`Health:` lines, a `Debt accepted:` PR line, `CODEOWNERS`, Golden Path wiring, measurement code) must not enter through this paragraph: a summary that links cannot retire gap records (option A was unbundled for that reason).

## The Fix

Append **one paragraph** to the **How it runs.** section of `ROADMAP.md`, after the existing paragraph, in the same voice (prose, no table, no dates), saying in order:

1. **The read:** the line as a whole is re-read as a synthesis of its receipts — the rows' `Row state:` lines, the debt map, the design sweeps, the transport axis — at every candidate cut, at every change to user obligations, layout or canonical ownership, and otherwise weekly; the synthesis is posted where the loop lives (D#19394 until its artifacts exist, then the owning epics) and the body's State line points at it.
2. **The pair:** the synthesis has a self-selected holder (for the v1 window: Clio, named on D#19394) and an independent reader of another family; a peer's acceptance — never presence or an old assignment — establishes the next reader; the holder hands off explicitly, and an unfilled seat names the action that fills it.
3. **The precondition:** the team must be reachable — *sent*, *mailbox-readable*, *route-active* and *delivered* are four different observations; the detect path is Brain #503, delivery Brain #30; no recurring read is scheduled over an unowned transport gap.
4. **Where a finding lands:** in the owning outcome or debt record, with its holder and next action (the acceptance tuple), never as a new tracker by default; `Row state:` summarizes and links; a merged repair, an accepted review or an updated row never passes the user outcome by itself — the installed walk does.
5. **What stays out:** `Health:` lines, debt lines in PR bodies, `CODEOWNERS` and route wiring return only when a failed transition names one and its consumer.

Link D#19394 and the first synthesis inline. Nothing else in the file changes.

**Design authority:** none challenged — the existing paragraph stays as written; this adds the line-level read beside the row-level one.
**Decision Record impact:** none (D#19394: `Decision Record: NOT_NEEDED`; the paragraph is procedure, not architecture).

## Acceptance Criteria

- [ ] AC-1 — `ROADMAP.md`'s **How it runs.** section carries one added paragraph that names, each in its own sentence: the recurring read and its triggers; the pair (holder + independent reader of another family) and the handoff rule; the transport precondition with its four observations and the two Brain homes; where a finding lands (the owning record, the acceptance tuple, `Row state:` summarizes and links); what stays out (the ungraduated carriers).
- [ ] AC-2 — The paragraph links D#19394 and the first synthesis comment (18854592); it names no date and no number that would stale.
- [ ] AC-3 — The diff touches `ROADMAP.md` only, adds no table, file, counter, label or workflow; the existing **How it runs.** paragraph is unchanged.
- [ ] AC-4 — The cross-family reviewer (a GPT seat, per the review routing for Fable PRs) answers from the paragraph alone, as a stranger: who re-reads the line, when, with whom, what must be true first, and where a finding goes — recorded in the review.
- [ ] AC-5 *(post-merge)* — D#19394's body line for artifact (ii) records the Institution half as done with the PR link; the engine half's ticket is filed and linked there.

## Out of Scope

The engine ROADMAP's paragraph (its own `neomjs/neo` ticket). The Brain (no release-line ROADMAP; the horizontal plan is #212 and its steward seat is D#19394 artifact (i)). Any `Health:`, debt-line, `CODEOWNERS` or Golden Path carrier. Changing the existing row-stewardship paragraph or the five-row table.

## Avoided Traps

- **A `Health:` line with today's numbers** — it stales the day after and cannot retire a gap record; D#19394 unbundled option A for exactly this.
- **A dated cadence** ("every Friday") — the triggers are events; the week is the fallback. A calendar beat never defers a known failure.
- **A new rule or skill** — D#19394 rejected option D (rule text has not changed a cadence yet); the paragraph tells a stranger how the line is kept honest, it does not instruct an agent per turn.
- **Naming the pair as an assignment** — the holder self-selected on D#19394; the reader seat is open by self-selection; the paragraph says how a seat is filled, not who must fill it.

## Related

D#19394 (the loop; synthesis 18854592) · #335 (the existing paragraph's origin, 09-30) · D#19384 (the Substrate paragraph's source) · #505 / #12 (the design and installed reads the synthesis consumes) · neomjs/neo-agent-brain#503, #30 (the transport axis) · neomjs/neo-agent-brain#212 (artifact (i)) · the engine line's sibling ticket (to be filed in `neomjs/neo`).

## Sweeps (ticket-create §1)

Live latest-open sweep: checked the latest 20 open issues of this repository at 2026-10-10 19:05:26Z (newest #666); no equivalent found. A2A in-flight claim sweep: the last 30 mailbox messages (all read-states, 17:46–19:02Z) carry no claim on this scope. Memory Core rationale sweep: my own 10-04 fold memory (5f5d570e) records artifact (ii) as "one 'How it runs' paragraph per release-line ROADMAP (Institution first)" — no prior filing. Own-assignment sweep: #505, #507, #351 — none touches `ROADMAP.md`. Structure map (§1c): N/A — a docs file, no `ai/` placement. §1d: D#19394 is at quorum; this ticket is the carrier of its artifact (ii), Institution half.

Origin Session ID: 9beaccd1-6dec-4d9e-b7b2-6a7b1a5d6d8e
Retrieval Hint: "META loop ROADMAP How it runs paragraph recurring read pair triggers transport precondition"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 9beaccd1-6dec-4d9e-b7b2-6a7b1a5d6d8e

## Timeline

- 2026-10-10T19:07:11Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T19:07:13Z @neo-fable-clio added the `documentation` label
- 2026-10-10T19:07:13Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T19:07:13Z @neo-fable-clio added the `ai` label
- 2026-10-10T19:10:42Z @neo-fable-clio cross-referenced by #19565
- 2026-10-10T19:11:51Z @neo-fable-clio cross-referenced by PR #668
- 2026-10-10T19:27:13Z @tobiu referenced in commit `fd7b947` - "docs(roadmap): the Institution line names how it is re-read (#667) (#668)

One paragraph after How it runs: the recurring synthesis and its triggers, the pair and its handoff rule, the transport precondition (Brain #503 / #30), where a finding lands, and the carriers that stay out — D#19394's graduation artifact (ii), Institution half."
- 2026-10-10T19:27:13Z @tobiu closed this issue

