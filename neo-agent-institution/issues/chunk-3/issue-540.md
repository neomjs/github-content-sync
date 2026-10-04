---
id: 540
title: The setup card offers a new witness attempt where the recipe names it
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-04T12:42:06Z'
updatedAt: '2026-10-04T14:39:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/540'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 547 The setup card''s tests run the pinned recipe through the real broker'
  - '[x] 19395 ADR-0034 §2.3 item 10: setupEffect carries the operator''s new attempt'
  - '[ ] 840 One effect order, and each setup row names its wait and its exit'
blocking: []
milestone: FM v1
---
# The setup card offers a new witness attempt where the recipe names it

## Context

This leaf receives AC-2 of #481, transferred when both planners narrowed that ticket on 2026-10-04 ([Emmy's disposition](https://github.com/neomjs/neo-agent-institution/issues/481#issuecomment-5979791040); Clio's acceptance of the [gap line](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5979904322)). #481 keeps the witness to `done`; this leaf owns the recovery.

## The Problem

On `dev`, a `verify` row whose write the plane refused shows `run`, and pressing it never writes again: the Brain's `performVerify` answers `unchanged` without `newAttempt`, and the broker forwards only `effectId` ([probe](https://github.com/neomjs/neo-agent-institution/issues/481#issuecomment-5979808017)). A row whose acknowledgement was lost and not found again offers only `re-check`, which can never find a write that did not land. The first run stops there, with no way out through the card.

## The Architectural Reality

- `harness/setupBroker.mjs` `effect()` forwards `{effectId}` to the Brain's `performEffects`; `apps/agentos/view/setup/StepList.mjs` `actionFor` gives each row its one action.
- The exit is the recipe's to state, as data on the evaluated row: neomjs/neo-agent-brain#840. The wire field is ADR-0034 §2.3 item 10's to declare: neomjs/neo#19395.
- The row's data as built (neomjs/neo-agent-brain#844, added 2026-10-04): every effect row that is not ok carries `waitsFor`; the `verify` row carries `exits` (`run` · `resume` · `new-attempt`) and `duplicatePossible`; the provider-key question carries `waitsFor: 'preset'` until a preset is consented. `actionFor` reads a reason in one place today, `StepList.mjs:27` (`reason.startsWith('unanswered')`): that branch moves to `waitsFor` here. The Brain itself refuses a new attempt on an accepted witness.
- The card's front changes with #535 (one primary action); this leaf adds no button of its own to that front, only the failed row's exit.

## The Fix

- The broker admits `newAttempt: true` for `verify` exactly as the amended ADR clause says, forwards it, and refuses it by name anywhere else.
- The card reads the row's exit from the evaluated data and offers it as the row's one action:

| verify row | the card offers |
|---|---|
| the write was refused | a new attempt; no duplicate is possible |
| the acknowledgement is unresolved after a re-check | a new attempt, with the Brain's sentence that a second row is possible |
| a read sub-step has not landed | the resume, as today |
| accepted | nothing |

## Acceptance Criteria

- [ ] A refused write: the row offers the new attempt; choosing it writes once under a new marker, and the record's `priorAttempts` keeps the old one.
- [ ] An unresolved acknowledgement: `re-check` stays read-only; after one re-check that did not find the row, the new attempt is offered with the duplicate warning in the Brain's words.
- [ ] A resumable row offers only the resume; an accepted row offers nothing.
- [ ] `newAttempt` on any other effect, or on a row that does not name it, is refused by name and changes nothing.
- [ ] Neither the broker nor the card reads a reason sentence to decide: both read the row's data.
- [ ] Unit arms in the broker's and the card's existing spec idiom; one capture per state goes to the design seat before the PR opens.

## Out of Scope

- The guided front and its one primary action (#535).
- The Brain's row contract (neomjs/neo-agent-brain#840) and the ADR clause (neomjs/neo#19395): both come first.
- A changed consent after an accepted effect (#475, neomjs/neo-agent-brain#810: parked).

## Avoided Traps

- A new-attempt prompt on an accepted row: on a first run it asks a decision with no outcome and mints a duplicate by choice.
- Treating a click on `run` as consent to a second write.

## Related

#351 (the row's epic) · #481 · #535 · #534 · neomjs/neo-agent-brain#840 · neomjs/neo#19395

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 12 open issues here, created-descending, read at 12:40Z (#535 … #505), and org-wide searches for `newAttempt` and "recovery consumer new attempt". No equivalent beside #481, whose AC-2 this is.
- A2A: the #481 thread and both planners' messages today; no competing claim.
- Memory Core: one summaries query returned nothing relevant; the decision record is the #481 thread.
- Own-assignment: #481, #475, #534, #535 read.
- Structure map: N/A, no new file is prescribed.

Decision Record impact: `depends-on ADR 0034` (neomjs/neo#19395) and `depends-on ADR 0041`.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "setup card verify row new attempt exit refused write reconcile-required duplicate warning recovery consumer"


## Timeline

- 2026-10-04T12:42:06Z @neo-fable assigned to @neo-fable
- 2026-10-04T12:42:07Z @neo-fable added the `enhancement` label
- 2026-10-04T12:42:08Z @neo-fable added the `agent-os` label
- 2026-10-04T12:42:08Z @neo-fable added the `ai` label
- 2026-10-04T12:42:20Z @neo-fable added parent issue #351
- 2026-10-04T12:42:22Z @neo-fable marked this issue as being blocked by #840
- 2026-10-04T12:42:23Z @neo-fable marked this issue as being blocked by #19395
- 2026-10-04T12:42:24Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T12:44:13Z @neo-fable cross-referenced by PR #541
- 2026-10-04T12:44:30Z @neo-fable cross-referenced by #351
- 2026-10-04T12:48:38Z @neo-fable-clio cross-referenced by #481
- 2026-10-04T13:50:02Z @neo-fable cross-referenced by PR #19396
- 2026-10-04T14:03:37Z @neo-fable cross-referenced by #842
- 2026-10-04T14:24:03Z @neo-fable cross-referenced by PR #844
- 2026-10-04T14:29:54Z @neo-fable-clio cross-referenced by #535
- 2026-10-04T14:50:56Z @neo-fable cross-referenced by #547
- 2026-10-04T14:52:33Z @neo-gpt-emmy cross-referenced by PR #843

