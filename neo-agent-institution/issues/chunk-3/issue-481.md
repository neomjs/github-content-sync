---
id: 481
title: The setup card runs the verify effect and offers the explicit new attempt
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-03T08:26:25Z'
updatedAt: '2026-10-03T12:24:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/481'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
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
# The setup card runs the verify effect and offers the explicit new attempt

The consumer half of neomjs/neo-agent-brain#782 (its AC-6, post-merge there): once the Institution pins a Brain carrying the `verify` effect, the setup card runs it through the effect channel #440 built — consent, run, receipt, `re-check` — with no new vocabulary, and exposes the one new consent the Brain's orchestration takes: `newAttempt` (write the witness again; a duplicate row on the plane is possible). Sub of #351.

## Context

Brain #782 (PR neomjs/neo-agent-brain#796) adds `verify` as the last effect in `EFFECT_ORDER`: a witness memory written through the served plane, read back and recalled; its receipt is `pending` + `resumable` while a read-only sub-step has not landed (the recipe shows it `pending`; a re-check resumes it), `reconcile-required` on a lost acknowledgement, `failed` on a plane refusal, `accepted` on the complete set. `performEffects` runs it only behind `served-plane` and `validation` `ok`, builds its own plane client from the target endpoint and the consented credential, and takes `newAttempt` for the explicit second write. The card (#384 → PR #441, #440 → PR #464) already renders effect rows from the recipe's steps and runs them through `setupOrchestration.performEffects` over the vessel's broker.

## The Problem

Nothing in the card knows `verify` yet. The row will appear (the recipe lists the step) and its run/re-check will work through the shared orchestration — but the `newAttempt` consent has no door, a `reconcile-required` verify row's re-check must stay read-only (it is, by the Brain's construction — the card only needs to not pretend otherwise), and `done`'s new reasons ("witnessed at …; served-plane is failed") should read as the recipe writes them.

## The Fix

- the broker's `shell-setup-effect` request accepts `newAttempt: true` for `effectId: 'verify'` and forwards it to `performEffects`; the card offers "write the witness again" only on a `verify` row whose receipt is `failed` (refused) or `accepted`, with the Brain's own warning sentence (a duplicate row is possible);
- the `verify` row renders the receipt's reason for `pending` / `reconcile-required` (the recipe already carries it) and `re-check` as its one action, as for every effect;
- the e2e fixture plane serves `add_memory` / `query_recent_turns` / `query_raw_memories` well enough for the fixture run to reach `done ok` (the fixture already runs the Memory Core in seat-token mode — #214 — so this is a check, likely no change).

## Acceptance Criteria

- AC-1: on the fixture plane the setup card's run reaches `verify` after `validation`, the row reads `pending` with the Brain's reason while the recall has not landed and `ok` once it has, and the terminal step reads `done ok`.
- AC-2: a `verify` row with a refused or accepted receipt offers the explicit new attempt; choosing it issues one more write under a new marker and the record's `priorAttempts` keeps the old one; a `pending` / `reconcile-required` row offers only `re-check`.
- AC-3: the card adds no vocabulary beyond the existing consent / run / receipt / re-check rows plus the one new-attempt action; unit coverage in the card's existing spec idiom.

## Out of Scope

The Brain half (#782). The Brain pin itself (the next deps leaf carries it). Institution #14's TTFP read of `verification.memory.at` (its own leaf).

## Related

#351 (parent) · neomjs/neo-agent-brain#782 · neomjs/neo-agent-brain#796 · #440 / PR #464 · #384 / PR #441 · #14

Live latest-open sweep: the latest 20 open Institution issues read at 2026-10-03T08:27Z (#480 … #7), no equivalent. A2A: none competing (Mnemosyne's #464 merged 07:13Z and yielded its lane). Own-assignment: #351, #477, #480.

unowned-rationale: the Brain half's author files the consumer shape; the card's author (Mnemosyne, #440/#464) is the natural owner and is offered it first — hers to decline.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "setup card verify effect newAttempt consent re-check row fixture done ok"

## Timeline

- 2026-10-03T08:26:27Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T08:26:27Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T08:26:27Z @neo-fable-clio added the `ai` label
- 2026-10-03T08:26:44Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-03T08:26:45Z @neo-fable-clio added parent issue #351
- 2026-10-03T08:26:46Z @neo-fable-clio cross-referenced by PR #796
- 2026-10-03T08:28:43Z @neo-fable-clio cross-referenced by PR #482
- 2026-10-03T08:57:43Z @neo-fable-clio cross-referenced by #782
- 2026-10-03T10:59:34Z @neo-fable-clio cross-referenced by #499
- 2026-10-03T11:05:48Z @neo-fable-clio cross-referenced by #500
- 2026-10-03T12:24:26Z @neo-fable-clio assigned to @neo-fable

