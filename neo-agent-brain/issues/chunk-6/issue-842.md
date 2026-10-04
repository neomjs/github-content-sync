---
id: 842
title: 'ADR 0041 §2.10: the evaluated setup row is the consumer''s contract'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-04T14:03:36Z'
updatedAt: '2026-10-04T15:04:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/842'
author: neo-fable
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
blocking:
  - '[ ] 840 One effect order, and each setup row names its wait and its exit'
closedAt: '2026-10-04T15:04:00Z'
milestone: FM v1
---
# ADR 0041 §2.10: the evaluated setup row is the consumer's contract

## Context

#840 makes the first-run recipe the one effect order and has each effect row name its wait and its exit as data. Its fifth criterion asks that ADR 0041 state that row contract "in its own PR first" (ADR 0005 §6.5: a decision record's revision is a separate PR, before the implementation). A pull request resolves exactly one ticket, so that PR needs this leaf: #840's fifth criterion is carved into it and replaced there by a pointer. No scope is added.

## The Problem

ADR 0041 §2.6 and §2.9 say what an effect may never do: re-run once accepted, replay an ambiguous one, write the witness twice without consent. They do not say what a consumer may read from an evaluated row. Three facts are therefore undecided in the record:

- which list orders the effects (the recipe lists `write-env` before `write-secrets`; the orchestration runs them the other way round);
- what a row that cannot run yet says;
- how a consumer learns a `verify` row's exit without matching its reason sentence.

Evidence: the probe on neomjs/neo-agent-institution#481 ([results](https://github.com/neomjs/neo-agent-institution/issues/481#issuecomment-5979808017)).

## The Architectural Reality

- `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`: §2 holds the decision (items 1–9), §3 the witness every implementing leaf inherits.
- §2.9 enumerates the host's own metadata of a witness attempt; a new stamp has to be named there.
- The contract's consumers: #840 (the recipe, the orchestration, the verify effect, the CLI), neomjs/neo-agent-institution#540 and neomjs/neo-agent-institution#535 (the setup card), and ADR 0034's `setupEffect` clause, which admits a new attempt "only when the Brain's evaluated row names a new attempt as its exit" (neomjs/neo#19396).

## The Fix

- §2 gains item 10, the row contract: the recipe's step list is the one order; an effect row that is not `ok` names what it waits for as data, and selecting an effect behind an unsettled one reports the wait and performs nothing past it; the `verify` row names its exits as data, with whether a duplicate is possible; no consumer reads a reason to choose an action.
- §2.9's list of host metadata gains the stamp of a reconciliation read that answered without the attempt's marker. That fact separates "search first" from "a new attempt is the remaining way on".
- §3 gains the contract's witness.

## Acceptance Criteria

- [ ] §2.10 states the one order, the wait as data with its report, and the `verify` row's exits for each state: not attempted; a read outstanding; the write refused; the acknowledgement lost and searched for without a find; accepted. The duplicate fact is stated per state.
- [ ] §2.10 states that a new attempt on an accepted witness is refused, that one consented outside the row's exits is still made, and that every attempt keeps the exits it was offered (added 2026-10-04 on the record author's read of the PR).
- [ ] §2.9 names the reconciliation stamp and the offered exits.
- [ ] §3 names the witness an implementing PR must carry.
- [ ] Merges before #840's implementation PR.

## Out of Scope

- The implementation (#840) and the card (neomjs/neo-agent-institution#540, neomjs/neo-agent-institution#535).
- Exits for host-effect rows: a failed host-effect row whose receipt is accepted is #810's subject.

## Avoided Traps

- A status enum for the exit: two exits can hold at once (resume, and a new attempt), so the row carries a list.
- Field names beyond the three the consumers are written against (`waitsFor`, `exits`, `duplicatePossible`).

## Related

#840 (the implementation, blocked by this) · #812 (the same record's §2.6 and §2.7 amendment, parked with #810) · #782 · neomjs/neo-agent-institution#351 · neomjs/neo#19396

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 20 open issues here, created-descending, read at 14:02Z and again at 14:03Z (#841 … #549), plus org-wide searches for "ADR 0041 row contract" and `waitsFor`. No equivalent; #812 amends other sections.
- A2A: all read-states, the last 30 messages (12:17Z to 14:00Z). No competing claim on ADR 0041 or the setup rows.
- Memory Core: one raw-memories query on the symptom (rows run in another order than shown, a run that does nothing, a failed verify whose need is unknown) returned the probe turn and #782's design turn. No prior decision on the row's data.
- Own-assignment: #840, #812, #810 read.
- Structure map: N/A, no file is created or moved.

Decision Record impact: `amends ADR 0041`.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "ADR 0041 row contract waitsFor exits duplicatePossible one effect order reconciliation stamp"


## Timeline

- 2026-10-04T14:03:36Z @neo-fable assigned to @neo-fable
- 2026-10-04T14:03:38Z @neo-fable added the `documentation` label
- 2026-10-04T14:03:38Z @neo-fable added the `enhancement` label
- 2026-10-04T14:03:38Z @neo-fable added the `ai` label
- 2026-10-04T14:03:38Z @neo-fable added the `architecture` label
- 2026-10-04T14:03:39Z @neo-fable added the `agent-os` label
- 2026-10-04T14:03:50Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T14:03:53Z @neo-fable cross-referenced by #840
- 2026-10-04T14:05:32Z @neo-fable cross-referenced by PR #843
- 2026-10-04T14:06:20Z @neo-fable cross-referenced by #351
- 2026-10-04T14:24:03Z @neo-fable cross-referenced by PR #844
- 2026-10-04T14:30:13Z @neo-fable referenced in commit `e6b2ef3` - "docs(adr): ADR 0041 §2.10 refuses a new attempt on an accepted witness and keeps the exits each attempt was offered (#842)"
- 2026-10-04T14:39:02Z @neo-fable referenced in commit `f4da4ac` - "docs(adr): ADR 0041 §2.10 names the wait of a question the preset decides (#842)"
- 2026-10-04T15:04:00Z @tobiu referenced in commit `9254cbf` - "docs(adr): ADR 0041 §2.10 states the setup row contract: one order, the wait and the exits as data (#842) (#843)

* docs(adr): ADR 0041 §2.10 states the setup row contract: one order, the wait and the exits as data (#842)

§2 gains item 10: the recipe's step list is the one order; an effect row that is not ok names what it waits for as data; the verify row names its exits as data with whether a duplicate is possible; no consumer reads a reason to choose an action. §2.9 names the reconciliation stamp; §3 gains the contract's witness.

* docs(adr): ADR 0041 §2.10 refuses a new attempt on an accepted witness and keeps the exits each attempt was offered (#842)

* docs(adr): ADR 0041 §2.10 names the wait of a question the preset decides (#842)"
- 2026-10-04T15:04:00Z @tobiu closed this issue

