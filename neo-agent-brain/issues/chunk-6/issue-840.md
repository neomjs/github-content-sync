---
id: 840
title: 'One effect order, and each setup row names its wait and its exit'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-04T12:41:27Z'
updatedAt: '2026-10-04T16:54:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/840'
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
  - '[x] 842 ADR 0041 §2.10: the evaluated setup row is the consumer''s contract'
  - '[x] 19395 ADR-0034 §2.3 item 10: setupEffect carries the operator''s new attempt'
blocking:
  - '[ ] 540 The setup card offers a new witness attempt where the recipe names it'
closedAt: '2026-10-04T16:54:17Z'
milestone: FM v1
---
# One effect order, and each setup row names its wait and its exit

## Context

Row 1 of FM v1 is the outside operator's first run (neomjs/neo-agent-institution#351). A probe drove the Institution's setup broker over this repository's own recipe, orchestration and verify modules at `5d466610`, with the outside world faked ([results](https://github.com/neomjs/neo-agent-institution/issues/481#issuecomment-5979808017)). It found three places where the recipe's consumer cannot know what an effect row needs. Both planners accepted this leaf on 2026-10-04 ([gap line](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5979904322)).

## The Problem

- **Two orders.** `RECIPE_STEPS` in `ai/services/fleet/firstRunRecipe.mjs` lists the effect rows as `write-env`, `write-secrets`, `compose-up`, `verify`. `EFFECT_ORDER` in `ai/services/fleet/setupOrchestration.mjs:34` is `write-secrets`, `write-env`, `compose-up`, `verify`. A consumer renders the first; the orchestration executes the second.
- **A silent halt.** `performEffects` walks `EFFECT_ORDER` and leaves the loop without a report at the first unsettled effect that is not the selected one. The caller gets the record back unchanged and no reason. In the setup card the top row's `run` answers ok and does nothing.
- **An exit nobody states.** A verify whose write the plane refused stays `failed`, and a plain run answers `unchanged` (`verifyEffect.mjs:155`); only `newAttempt` moves it. A verify whose acknowledgement is lost and whose marker is not found stays `reconcile-required`, and its reason says "consent to a new attempt writes a second row". The evaluated step carries `status` and `reason`, so a consumer could learn the exit only by matching that prose.

## The Architectural Reality

- `firstRunRecipe.mjs` owns the steps and their evaluation; `setupOrchestration.mjs` owns execution; `verifyEffect.mjs` owns the witness and its outcomes (`written`, `adopted`, `resumed`, `refused`, `reconcile-required`, `unchanged`).
- Consumers: the CLI (`ai/scripts/setup/firstRun.mjs`) and the Institution's broker and setup card.

## The Fix

- **One order.** The orchestration reads the recipe's effect steps; `EFFECT_ORDER` goes, or is derived from them.
- **A waiting effect says so.** Selecting an effect behind an unsettled one reports which effect it waits for, and the waiting row's evaluation carries the same fact as data.
- **The verify row states its exit as data:** resume (a read sub-step has not landed), a new attempt with no duplicate possible (the write was refused), a new attempt with a possible duplicate (the acknowledgement is lost and a re-check did not find the row), or none (accepted).

## Contract Ledger

Added 2026-10-04 on the reviewer's request (neomjs/neo-agent-brain#844): the surfaces this leaf hands to consumers. Authority for every row: ADR 0041 §2.10, with §2.9 for the two attempt fields.

| Target surface | Behaviour | Fallback for a consumer on an older Brain | Consumers | Evidence |
|---|---|---|---|---|
| `RECIPE_STEPS` order; `EFFECT_ORDER`, `VERIFY_GATES` derived from it | the effects run in the listed order: `write-secrets`, `write-env`, `compose-up`, `verify`; no second list | none needed: the export keeps its name | the CLI; the Institution's broker (`EFFECT_ORDER.includes`) | `setupOrchestration.spec.mjs`, "AC-2: with no filter …" |
| effect row `waitsFor` (step id or `null`) | the first earlier question, effect or declared gate that is not `ok`; `null` on an `ok` row and on one that can run | absent reads as `null` | the card (neomjs/neo-agent-institution#540, neomjs/neo-agent-institution#535's "Run next step"); the CLI's text | `firstRunRecipe.spec.mjs`, "an effect row that is not ok names the step it waits for …" |
| provider-key question row `waitsFor: 'preset'` | present until a preset is consented | absent reads as an answerable question | the card's `actionFor` | `firstRunRecipe.spec.mjs`, the provider-key assertions in "a missing observer is unknown …" |
| verify row `exits` (list of `run`, `resume`, `new-attempt`) and `duplicatePossible` | `[run]` not attempted · `[resume]` a read outstanding or refused, or a lost acknowledgement not yet searched · `[new-attempt]` after a refused write · `[resume, new-attempt]` with `duplicatePossible: true` after a search without a find · `[]` accepted | absent: the consumer offers what the status alone offered before, and no new attempt | the card (neomjs/neo-agent-institution#540); the shell's admission of `newAttempt` (ADR 0034 §2.3 item 10); the CLI's text | `verifyEffect.spec.mjs`, the five arms under "the row names its exits as data" |
| record `verification.attempt.searchedAt` | stamped when a reconciliation read answered without the marker | absent reads as not searched | `verifyExits` | `verifyEffect.spec.mjs`, "the acknowledgement lost: …" |
| record `verification.attempt.offered` | the exits the row named when the attempt was consented | absent on an attempt an older Brain made | a reader of the record | `verifyEffect.spec.mjs`, "the write refused: …"; `setupOrchestration.spec.mjs`, the last arm |
| `performEffects` report `'<effect>' waits for '<step>': it is <status> (<reason>)` | for a selection behind an unsettled effect, behind an unanswered question, and for `verify` behind its gates | the old run returned without a word | the CLI's stderr; the broker's refusal reason | `setupOrchestration.spec.mjs`, the two selection arms and "an effect that waits for a question says so …" |
| a new attempt on an accepted witness | refused by the effect and reported: `'verify' is accepted: no new attempt is made, …` | the old run wrote a second row | the CLI's `--new-attempt`; the shell | `verifyEffect.spec.mjs`, "accepted: …"; `setupOrchestration.spec.mjs`, the last arm |
| the CLI's text row suffix `· waits for <step>` and `· next: …` | read from the row's fields | a row without the fields prints no suffix | the operator at a terminal | `firstRun.spec.mjs`, "the text output reads what comes next from the row's data …" |
| a host-effect row whose receipt is `failed` (folded in on 2026-10-04, neomjs/neo-agent-institution#351 comment 5981959620) | reads `failed` with the receipt's reason; a run retries it | the old row read `pending` with the observer's sentence | the card; the CLI | `firstRunRecipe.spec.mjs`, the arm this criterion adds |

## Acceptance Criteria

- [ ] The recipe's effect steps and the execution order are one list; a test fails if they diverge.
- [ ] Selecting an effect behind an unsettled one reports "`<effect>` waits for `<effect>`" and changes nothing; the waiting row's evaluation carries that fact as data.
- [ ] The evaluated verify step names its exit for each of the four states above, including whether a duplicate is possible; one unit arm per state over `performVerify`'s own outcomes.
- [ ] No consumer needs a reason sentence to choose an action: the CLI's own output reads the new fields.
- [ ] A new attempt on an accepted witness is refused and reported; one consented outside the row's exits is still made, and every attempt keeps the exits it was offered (added 2026-10-04 on the record author's read of #842's PR).
- [ ] A host-effect row whose receipt is `failed` reads `failed` with the receipt's reason, never `pending` with the observer's sentence (folded in 2026-10-04 by the planner, neomjs/neo-agent-institution#351 comment 5981959620).
- [ ] The row contract's record is its own leaf, #842 (ADR 0041 §2.10; carved out 2026-10-04 because a pull request resolves one ticket); this leaf's PR merges after it.

## Out of Scope

- The card's actions: the Institution's recovery consumer and the guided front (neomjs/neo-agent-institution#535).
- The `setupEffect` wire field: the Engine's ADR-0034 §2.3 item 10 clause, which lands with or before this contract.

## Avoided Traps

- Letting a consumer treat a click on one row as permission to run another.
- Parsing reasons.
- A second order "for display".

## Related

neomjs/neo-agent-institution#351 · neomjs/neo-agent-institution#481 · #782 (the verify effect) · #810 (parked: a changed consent re-applies) · neomjs/neo-agent-institution#535

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 12 open issues here, created-descending, read at 12:40Z (#837 … #700), and org-wide searches for `EFFECT_ORDER`, `newAttempt`, `setupEffect` and "next action waiting effect". No equivalent.
- A2A: the #481 thread and both planners' messages today; no competing claim.
- Memory Core: one summaries query on the effect order and the verify exit returned nothing relevant. The decision record is the #481 thread.
- Own-assignment: #810, #812 read; they concern a changed consent, not the order or the exit.
- Structure map: N/A, no new file is prescribed.

Decision Record impact: `depends-on ADR 0041`; the row contract may amend it.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "setup recipe one effect order EFFECT_ORDER silent halt verify exit new attempt duplicate possible evaluated row data"




## Timeline

- 2026-10-04T12:41:27Z @neo-fable assigned to @neo-fable
- 2026-10-04T12:41:31Z @neo-fable added the `enhancement` label
- 2026-10-04T12:41:31Z @neo-fable added the `ai` label
- 2026-10-04T12:41:32Z @neo-fable added the `agent-os` label
- 2026-10-04T12:41:47Z @neo-fable cross-referenced by #19395
- 2026-10-04T12:42:07Z @neo-fable cross-referenced by #540
- 2026-10-04T12:42:20Z @neo-fable added parent issue #351
- 2026-10-04T12:42:22Z @neo-fable marked this issue as blocking #540
- 2026-10-04T12:42:24Z @neo-fable marked this issue as being blocked by #19395
- 2026-10-04T12:42:26Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T12:44:13Z @neo-fable cross-referenced by PR #541
- 2026-10-04T12:44:30Z @neo-fable cross-referenced by #351
- 2026-10-04T12:48:38Z @neo-fable-clio cross-referenced by #481
- 2026-10-04T13:50:02Z @neo-fable cross-referenced by PR #19396
- 2026-10-04T14:03:37Z @neo-fable cross-referenced by #842
- 2026-10-04T14:03:46Z @neo-fable marked this issue as being blocked by #842
- 2026-10-04T14:05:32Z @neo-fable cross-referenced by PR #843
- 2026-10-04T14:14:02Z @neo-fable-clio cross-referenced by #52
- 2026-10-04T14:24:03Z @neo-fable cross-referenced by PR #844
- 2026-10-04T14:29:45Z @neo-fable referenced in commit `f4b24c1` - "feat(fleet): an accepted witness refuses a new attempt, and each attempt keeps the exits it was offered (#840)

performVerify makes no new attempt once the witness is accepted, and the orchestration says so before it builds a plane client. A new attempt consented while the row did not name one is still made; every attempt records the exits the row named when it was consented (attempt.offered)."
- 2026-10-04T14:38:43Z @neo-fable referenced in commit `4f045d5` - "feat(fleet): the provider-key question says as data that it waits for the preset (#840)

The Institution's card chose that row's action by the reason's first word; the row now carries waitsFor: 'preset' until a preset is consented, so no consumer has to."
- 2026-10-04T14:40:38Z @neo-fable cross-referenced by #534
- 2026-10-04T16:19:45Z @neo-fable referenced in commit `4f5ff57` - "feat(fleet): a host effect whose last attempt failed reads failed with the receipt's reason (#840)

The row read pending with the observer's sentence, so a failed write-env told nobody why. The absent result is still what decides; the failed receipt beside it says why it is absent, the way an accepted receipt already says a result is gone. A run retries it, and nothing turns green from a receipt."
- 2026-10-04T16:22:30Z @neo-fable cross-referenced by #848
- 2026-10-04T16:47:38Z @neo-fable-clio cross-referenced by #850
- 2026-10-04T16:50:31Z @neo-fable-clio cross-referenced by PR #852
- 2026-10-04T16:54:17Z @tobiu referenced in commit `4f28585` - "feat(fleet): the first-run recipe is the one effect order, and each row names its wait and its exits (#840) (#844)

* feat(fleet): the first-run recipe is the one effect order, and each row names its wait and its exits (#840)

EFFECT_ORDER is read from the recipe's steps; the recipe lists the secret files before the carrier, as every run has written them. An effect row that is not ok carries waitsFor, and a selected effect behind an unsettled effect or an unanswered question reports that wait instead of returning silently. The verify row carries exits and duplicatePossible (verifyExits); a reconciliation read that answered without the marker stamps attempt.searchedAt, after which a new attempt is named beside the re-check. The CLI's text output reads the new fields.

* feat(fleet): an accepted witness refuses a new attempt, and each attempt keeps the exits it was offered (#840)

performVerify makes no new attempt once the witness is accepted, and the orchestration says so before it builds a plane client. A new attempt consented while the row did not name one is still made; every attempt records the exits the row named when it was consented (attempt.offered).

* feat(fleet): the provider-key question says as data that it waits for the preset (#840)

The Institution's card chose that row's action by the reason's first word; the row now carries waitsFor: 'preset' until a preset is consented, so no consumer has to.

* feat(fleet): a host effect whose last attempt failed reads failed with the receipt's reason (#840)

The row read pending with the observer's sentence, so a failed write-env told nobody why. The absent result is still what decides; the failed receipt beside it says why it is absent, the way an accepted receipt already says a result is gone. A run retries it, and nothing turns green from a receipt."
- 2026-10-04T16:54:17Z @tobiu closed this issue

