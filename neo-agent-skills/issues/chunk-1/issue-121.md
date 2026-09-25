---
id: 121
title: 'A ticket that calls a designed behavior wrong carries no design authority, and review trusts it'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-fable
createdAt: '2026-09-25T20:13:12Z'
updatedAt: '2026-09-25T21:35:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/121'
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
closedAt: '2026-09-25T21:32:50Z'
---
# A ticket that calls a designed behavior wrong carries no design authority, and review trusts it

## Context

Specimen, 2026-09-25 (neomjs/neo#19225 → PR #19231, closed unmerged by the author): a capture-mode receipt showed a dragged vessel's content sitting one title bar below the pointer on a real window. The author (me) filed the placement as a defect, implemented "the content follows the pointer", passed a unit arm, a green unit suite and a green headed film run, opened the PR and broadcast the take as unblocked. The operator caught it at review: the window's top-left corner riding the pointer is the design — ADR 0029 §2.8.6, pointer-follow "at the live logical drag origin" — because with the pointer at the corner the drop targets around it stay uncovered. Operator, verbatim: "you derail, fail VBA and create tickets which make neo worse than before. then you spend time and tokens to implement negative ROI lanes. then peers would review it, TRUST your ticket and approve it. i do not have time to babysit 8 peers at all times and i might miss the reject a PR like this one. time for friction->gold."

The ticket's Architectural Reality cited file:line for the follow and the metric and never the record that defines the behavior as intended. Nothing in `ticket-create` asks for that record when a ticket calls an existing behavior wrong, and nothing in `pr-review` halts on its absence: the Premise Snapshot names "source-of-authority substrate" in general, and §9.0's triggers name a false premise without saying how a reviewer would see one that the ticket itself asserts.

## The Problem

A ticket is the premise a PR review inherits. When it declares a designed behavior a defect, the reviewer's Patch-Blind Premise Snapshot reads the ticket as the authority, the diff matches the ticket, the tests are green, and the approval follows. The class is specific: **behavior called wrong from a receipt** — geometry, timing, a rendered frame — where no crash, no red, and no contradiction with the code's own JSDoc exists. A receipt shows what happens; it never shows whether that is wanted. The only thing that answers "wanted?" is the record that decided it (an ADR section, a guide, the owning module's JSDoc), and the workflow never makes the author read it before the Fix is written. The cost when it slips: an author's session, two reviewers' rounds, and the operator as the last line, which the operator has said cannot scale.

## The Architectural Reality

- `ticket-create/references/ticket-create-workflow.md` §0 ("Understand the intent before you write the ticket") asks for intent from JSDoc / `ask_knowledge_base` in general; §5's **The Architectural Reality** bullet asks for file:line; §8's anti-pattern table has "Quotation as root cause" but no row for a receipt as a design verdict.
- `pr-review/references/pr-review-guide.md` §9.0 (Cycle-1 Premise Pre-Flight) lists "false premise" among the triggers; `pr-review/assets/pr-review-template.md` Premise Snapshot lists "source-of-authority substrate" among the inputs. Neither names the ticket's own design-authority citation as the thing to check.
- Owning directories: `.agents/skills/ticket-create/`, `.agents/skills/pr-review/` (the two skills whose payloads the gate joins).

## The Fix

1. **ticket-create — the Design-authority line.** §0 gains the rule: a ticket that calls an existing behavior wrong (not a crash, not a red, not a contradiction with the surface's own JSDoc) MUST carry, in The Architectural Reality, a `Design authority:` line quoting the record sentence that defines the current behavior as intended — an ADR section, a guide, the owning module's JSDoc — or `Design authority: none found` with the searches run (the ADR directory, the guides, `ask_knowledge_base`). Without a found sentence the ticket is a design question and is filed as a fork on the owning record (a comment on the ADR's epic or a Discussion), never as a defect with a Fix. §5's Reality bullet names the line; §8 gains the row "Receipt as design verdict — a receipt shows what happens, never whether it is wanted".
2. **pr-review — the premise check has a handle.** §9.0 gains the trigger: a behavior-change PR whose close-target ticket carries no `Design authority:` line, or whose line contradicts the diff, is premise-unverified → Request Changes with one RA (cite the record or file the fork), never an approval on tests and green. The template's Premise Snapshot lists "the ticket's `Design authority:` line and the record it cites" among the inputs read before the patch.
3. **Accretion defense / sunset.** Both are prose gates; the retirement trigger is a body lint on the `Design authority:` line for tickets that name a behavior change (out of scope here, a later leaf under #86's class). The added bytes are one paragraph, one bullet fragment, one table row, one trigger sentence and one snapshot input.

## Acceptance Criteria

- [ ] AC-1 `ticket-create/references/design-authority.md` carries the rule — the trigger keyed on the observable (a ticket that calls an existing behavior wrong), the three sources, the none-found fork, the sunset — reached from `ticket-create-workflow.md` §0 by a one-line pointer. *(Amended 2026-09-25, PR #122 review round 1: the corpus budget refused the in-file paragraph — the workflow file sits at the 25 000-byte per-file payload budget — so the rule loads on its trigger instead; the §5 sentence and the §8 row are folded into the reference.)*
- [ ] AC-2 `pr-review-guide.md` §9.0 names the missing-or-contradicting Design-authority line as a premise trigger with the one-RA disposition; `pr-review-template.md`'s Premise Snapshot lists the line among its inputs.
- [ ] AC-3 The PR body names the sunset condition (the body lint) as the gate's retirement trigger.
- [ ] AC-4 Post-merge: the next `ticket-create` invocation on a behavior-change ticket in neo shows the line in the body (any seat's ticket; cited here).

## Out of Scope

The body lint that would mechanize the line (a later leaf); changes to ADR 0029 or the dock behavior the specimen touched (neomjs/neo#19225 is restated to its driver-only minimum).

## Related

#62 (the sweeps never query closed state), #59 / #60 (body currency), #86 (the AC table is what the merge gate reads); neomjs/neo#19225, neomjs/neo#19231; ADR 0029 §2.8.6.

Live latest-open sweep: checked the latest 20 open issues of this repository at 2026-09-25T20:11:52Z; no equivalent (#62 / #59 / #60 / #86 are neighbours, none carries the design-authority gate). A2A in-flight sweep: no claim. Memory Core sweep: my own specimen record (session 4c0a5550, 20:06Z). Own-assignment sweep: none of mine covers it.

Retrieval Hint: `query_raw_memories("design authority line ticket-create behavior called wrong from a receipt PR #19231 withdrawn")`

Origin Session ID: 4c0a5550-17ba-4752-9852-846afa537c86

## Timeline

- 2026-09-25T20:13:12Z @neo-fable assigned to @neo-fable
- 2026-09-25T20:13:14Z @neo-fable added the `enhancement` label
- 2026-09-25T20:13:14Z @neo-fable added the `ai` label
- 2026-09-25T20:14:24Z @neo-fable cross-referenced by PR #122
- 2026-09-25T20:55:19Z @neo-fable referenced in commit `28a27bd` - "chore(release): bump to 0.1.21, every PR bumps because every merge publishes (#121)"
- 2026-09-25T21:03:26Z @neo-fable referenced in commit `63c0395` - "docs(skills): the design-authority rule loads behind a §0 pointer, inside the corpus budgets (#121)

[skill-growth-justified: a premise gate for tickets that call a designed behavior a defect — neo PR #19231 was nearly approved on such a ticket; the rule loads only on its trigger through a one-line §0 pointer; sunset = a body lint on the line]"
- 2026-09-25T21:20:10Z @neo-fable referenced in commit `d8b9bd3` - "docs(skills): a ticket that calls a designed behavior wrong carries its design authority, and review reads it (#121)

ticket-create §0 gains the Design-authority line: a ticket that declares an existing behavior a defect with no crash, no red and no JSDoc contradiction quotes the record sentence that defines the behavior as intended (ADR section, guide, owning JSDoc) or writes none-found with the searches run and files a fork on the record instead of a defect; §5's Reality bullet names the line, §8 gains the receipt-as-design-verdict row. pr-review §9.0 gains the handle: no line, or a line the diff contradicts, is a visible false premise, one RA, never an approval on tests and green; the template's Premise Snapshot reads the line. Sunset: a body lint on the line retires the prose gate. Specimen: neomjs/neo#19225 / PR #19231."
- 2026-09-25T21:20:10Z @neo-fable referenced in commit `ba9a483` - "chore(release): bump to 0.1.21, every PR bumps because every merge publishes (#121)"
- 2026-09-25T21:20:10Z @neo-fable referenced in commit `a04ea80` - "docs(skills): the design-authority rule loads behind a §0 pointer, inside the corpus budgets (#121)

[skill-growth-justified: a premise gate for tickets that call a designed behavior a defect — neo PR #19231 was nearly approved on such a ticket; the rule loads only on its trigger through a one-line §0 pointer; sunset = a body lint on the line]"
- 2026-09-25T21:20:10Z @neo-fable referenced in commit `e70f6e5` - "docs(skills): the design-authority trigger keys on the observable, the reviewer verdict is stated once, intake reads the line first (#121)

Review round 1 (Ada) RA-1: the reference fired on the conclusion ('declares a designed behavior a defect') instead of the observable ('calls an existing behavior wrong'); pr-review §9.0 regained #121's scope and the unverified-vs-false verdict classes; the reference's second paragraph is now a pointer to §9.0. Plus the suggested ticket-intake step-3 pointer."
- 2026-09-25T21:32:51Z @tobiu referenced in commit `5b4d9a8` - "Merge pull request #122 from neomjs/fix/121-design-authority-line

docs(skills): a ticket that calls a designed behavior wrong carries its design authority, and review reads it (#121)"
- 2026-09-25T21:32:51Z @tobiu closed this issue
- 2026-09-25T21:35:35Z @neo-fable cross-referenced by #123
### @neo-fable - 2026-09-25T21:35:59Z

Post-merge (PR #122 → dev 5b4d9a8, 2026-09-25 21:32Z):

- **AC-4 evidence:** the first ticket restated through the gate carries the line — neomjs/neo#19225, "Design authority: ADR 0029 §2.8.6 — 'Its admission reads the source's live outer extent and the target's live inner extent …'" — and its PR (neomjs/neo#19232) was reviewed on that handle: the cross-family seat falsified the quoted sentence against the implementation and found a pre-existing record-vs-code divergence the ticket's premise had hidden. The gate did what it was filed for, in its first use.
- **Sunset leaf filed:** #123 — the body lint that retires the reference's prose sunset, carrying Ada's round-1 note (a ticket without the line must cite the crash, the red or the JSDoc sentence instead; otherwise the lint only enforces where the author already agreed).

— Mnemosyne (Claude Fable 5.1, Claude Code). Session 4c0a5550-17ba-4752-9852-846afa537c86


