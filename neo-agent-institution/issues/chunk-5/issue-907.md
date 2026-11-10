---
id: 907
title: Accounts shows two read-back rows that nothing reads back
state: OPEN
labels:
  - bug
  - good first issue
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-10T19:04:52Z'
updatedAt: '2026-11-10T19:05:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/907'
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
milestone: FM v1
---
# Accounts shows two read-back rows that nothing reads back

## Context

A design sweep of the installed Fleet Manager on 2026-11-10 read the Accounts view on the team's own plane ([receipt](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6776500148), section `accounts`, Institution `86ca206d`, window 1966 × 1290 CSS px). Two findings share this leaf because they share one view and one small change set.

## The Problem

1. **Two rows wait for a reader that does not exist.** The config card's section "Operations · read back" renders Hooks and Wake subscriptions from `record.hooksActive` and `record.wakeSubscriptionsActive`. Both are fields of `AgentOS.model.AgentDefinition` (lines 102 and 106) and nothing writes them: `git grep` finds no other reference in the Institution's `apps` and `harness` at `86ca206d`, and none in `ai` of neomjs/neo-agent-brain at `67537f93`. Every one of the 13 accounts therefore shows `Not read back yet` twice, including seats that have been working for hours. "Yet" promises a read that the product does not perform.
2. **Two cards share one narrow column beside an empty area.** The detail area is 1676 px wide. The config card and the repositories card are both capped at 28rem (448 px) and stacked, so 1,228 px stay empty while the repositories card ends 235 px below the fold and the reader scrolls.

## The Architectural Reality

- `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs`: the section at line 546, `createToggleRow()` at line 705, and the class JSDoc, lines 33 to 44.
- `resources/scss/src/apps/agentos/accounts/Panel.scss`: `.fm-accounts-detail` and the cap with its reason, "no field spans the whole detail (the operator's 2026-09-28 ruling on full-width username and repository fields)".
- Design authority, rows: the class JSDoc, "The operational toggles are READ BACK — what the fleet actually observed." A row with no observer has nothing to read back, and `ROADMAP.md` row 2 asks of every state "its reason and the next step".
- Design authority, cards: the views epic's second question, "Does it have room — by DEFAULT?". The cap rules a field's measure, not the number of columns; the design seat's ruling is in the receipt, section `accounts`, point 5.

## The Fix

- The section renders a row only when its value was read back (`true` or `false`). With no row to show, the section and its heading do not render. The two fields and `createToggleRow()` stay for the producer that will fill them.
- `.fm-accounts-detail` seats the config card and the repositories card side by side when the detail is at least two card widths plus the gap wide, and stacks them below that. The add form keeps its single measure.

## Acceptance Criteria

- [ ] AC-1 With both fields `null` the card shows no "Operations" section; with one boolean it shows that row only. A unit arm covers both.
- [ ] AC-2 In a detail area of at least two card widths plus the gap, the two cards share a row, and an account with nine repositories needs no vertical scroll at 1966 × 1290. Below that width they stack. The arm asserts containment or geometry against the detail container.
- [ ] AC-3 The add form's measure is unchanged.
- [ ] AC-4 Post-merge: a design-sweep receipt on the next installed candidate records both.

## Out of Scope

A producer for the two values: that is a feature across the Brain and the harness, to be filed when a journey asks for it. The harness chips' edit behaviour, the add form's memory labels, and the sort order of the list (#862).

## Avoided Traps

- Rewording `Not read back yet` to a softer sentence. The row has no source; any sentence about its state is a guess.
- Widening the cap. The operator ruled on field width; two columns keep the measure and use the room.

## Related

#505 (the views epic; its sub-issue list is full, so this leaf links by reference) · #245 and #408 (the view's layout and the repository lines, both closed; neither excludes this) · #885 (later Accounts work).

Live latest-open + closed sweep: checked the latest 20 open issues and closed searches ("Catch up", "Chat rail placeholder", "Accounts card read back hooks", "side by side 28rem"; Out of Scope read on #245) at 2026-11-10T19:03Z; no equivalent or prior exclusion found. Backlog band: 47 open, orange; #885 is later scope and does not own these two defects. A2A claim sweep over the last 40 minutes at 19:03Z: none on this scope. Memory Core rationale sweep on the symptom: no prior decision. Own assignments (#505, #507, #351, #889): #505 is the parent by reference.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 0c3a6369-4269-4b0b-8ad5-0c9441248925

## Timeline

- 2026-11-10T19:04:54Z @neo-fable-clio added the `bug` label
- 2026-11-10T19:04:54Z @neo-fable-clio added the `good first issue` label
- 2026-11-10T19:04:54Z @neo-fable-clio added the `agent-os` label
- 2026-11-10T19:04:55Z @neo-fable-clio added the `ai` label
- 2026-11-10T19:04:55Z @neo-fable-clio added the `design` label
- 2026-11-10T19:05:57Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-11-10T19:07:33Z @neo-fable-clio cross-referenced by #505

