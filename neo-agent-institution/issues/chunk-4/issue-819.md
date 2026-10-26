---
id: 819
title: Confirm admitted seat family from Agent Detail
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T08:31:09Z'
updatedAt: '2026-10-26T08:31:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/819'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 700
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1298 Require the admitted binding before managed seat usability'
blocking: []
---
# Confirm admitted seat family from Agent Detail

## Context

The [D811 C8 contract](https://github.com/neomjs/neo-agent-institution/discussions/811) and [accepted delivery map](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190367) require an operator-facing manual confirmation path. This is the Institution consumer under Brain #700. It joins Brain #1296 (admitted source/readers), #1297 (typed Fleet publisher) and #1298 (host usability), while retaining the installed witness on #414.

## The Problem

An operator outside Neo's static roster needs to confirm the family of a seat they operate, see what was admitted, and understand why an identity remains unclassified. A generic “no family” label hides materially different remedies: a newly unconfirmed seat, disagreement across seats, incomplete membership, proposed binding or pending retirement.

Design authority: C8 acknowledgment AC A requires each named state to show affected seats, actual cause and remedy. Add Agent remains name + one PAT; neither a reported model nor a saved model declaration writes canonical family in this manual v1 slice.

## The Architectural Reality

`apps/agentos/view/fleet/detail/Container.mjs` composes the existing Configuration groups. `SeatModelContainer.mjs` renders declared/configured/observed model facts; `detail/Controller.mjs:onDeclareSeatModel` uses `ConfigIntentRoundTrip` and current seat-binding checks. `ParticipationContainer.mjs` shows a separate identity-wide fact; Brain #28 owns its future Bench/Return actions.

Reuse the existing container/button/form-field grammar, controller ownership and Store/Model readback. The generic model-save path is not family-write authority. Brain #875's graph-sourced Detail outcome remains related; this is one consumer, not a second identity settings system.

## The Fix / operator journey

1. Keep Add unchanged. In Agent Detail's Configuration, show the admitted family/provenance separately from model intent and observations. Offer explicit confirmation through Brain #1297's typed operator method, with an optional bound proposal where one is actually supported. Without one, use the same action with literal model null.
2. Show exact known family values and a deliberate new-family act; server validation owns collision/authority rules. Do not infer a family from harness, login, vendor aliases or a client-side catalog guess.
3. Retain one opaque retry token for one pending act; pass expected revisions and the exact selected binding. Show pending, admitted/idempotent, conflict and unavailable readback. A new click after losing the token is a new act, not claimed exactly-once recovery.
4. Render `unconfirmed-seat`, `ambiguous-seats`, `membership-unknown`, `proposed` and `retirement pending` from the source DTO, naming affected seats and the available in-FM remedy. Source-provided authority determines who can confirm/correct. Do not substitute a host script or credential question.
5. Preserve selected-seat/plane/incarnation fencing during requests and canonical Store writes. A late result cannot paint another seat or imply that local pending state is accepted.
6. Display correction history and its limits: the original confirming principal can correct only an allowed target in its tenure. First confirmation is never backdated. An unclassified author interval remains uncertifiable; a permitted audited correction or a PR created after that interval supplies the documented exit.

## Contract Ledger

| Surface | Authority | Behavior / fallback | Docs / evidence |
|---|---|---|---|
| Detail manual family action | Brain #1297 admitted operator method | One explicit act, exact family/model-null/revisions/token; no generic-save authority | Operator journey docs; actual wire/controller tests |
| Family and membership state | Brain #1296 complete projection | Five distinct refusal/pending states with seats, cause and remedy; unavailable never empty success | Source DTO matrix + rendered checks |
| Pending act/readback | Fleet durable intent + MC receipt | Canonical Store update and binding-scoped feedback; retry preserves token | Delayed responses, seat/plane switch and lost-ack controls |
| Historical correction display | Admitted provenance/target rules | Preserve replaced fact; no prior-owner rewrite or backdated first confirmation | Two-operator and historical interval UI controls |

## Acceptance Criteria

- [ ] AC-1: an outside-roster seat's operating principal can make an explicit manual confirmation from Detail, with and without a bound proposal; no proposal sends explicit null. Add remains name + one PAT, and model save/Start/read-back cannot activate a family.
- [ ] AC-2: pending, accepted/idempotent, conflict, refusal and unavailable outcomes are shown from canonical readback; one pending act retains its token and expected binding/revisions. Lost token is not presented as the same act.
- [ ] AC-3: all five C8 states render distinct cause, affected seat(s) and remedy. Exercise two operators, same-owner conflicting seats, incomplete baseline, proposed bind and pending removal. A roster classification remains authoritative and distinct from an unconfirmed declaration.
- [ ] AC-4: ordinary seat/non-owner/transfer denials remain meaningful. Correction names its prior record and reason, retains provenance and offers no predecessor rewrite or backdated first confirmation. Historical author refusal and its bounded exit are explicit.
- [ ] AC-5: selected seat/plane/incarnation changes and component destruction fence late feedback; Store/Model updates follow existing canonical arbitration. No stale answer becomes another seat's successful confirmation.
- [ ] AC-6: meaningful controller/wire tests plus rendered narrow/wide Detail checks pass; keyboard labels and pending controls remain usable. Styles stay in SCSS, providers at view roots, data on the existing Store/Model path.
- [ ] AC-7: documentation distinguishes admitted family, proposed/observed model and effective runtime. Source/UI evidence does not close #414's installed outside-roster required REQUEST_CHANGES journey.

## Discussion Criteria Mapping

C7 explicit act/null/retry → AC-1/2; C8 acknowledgment AC A → AC-3; provenance/tenure/temporal policy → AC-4; stable binding and UI lifecycle → AC-5; product evidence boundary → AC-6/7.

## Out of Scope / Avoided Traps

No new family policy, client-inferred authority, host command, credential editor, class-9 envelope, provisioning UI, automatic activation or replacement of Brain #28's participation work. This adds one manual consumer of the admitted source, not another ledger. Keep mechanics/state labels in understandable product language; protocol field names belong in developer docs.

Related: neomjs/neo-agent-brain#700, neomjs/neo-agent-brain#875, neomjs/neo-agent-brain#28, neomjs/neo-agent-brain#1296, neomjs/neo-agent-brain#1297, neomjs/neo-agent-brain#1298, #818, #414

## Decision Record impact

Depends on Institution ADR0038 C7/C8 amendments (#816 and #818 in that repository) and Brain ADR0041's class-9 relation (neomjs/neo-agent-brain#1294); aligned with ADR0019 and ADR0006's admitted-authority boundary. These are merge gates, not evidence that runtime delivery is complete.

Decision Record: **Required — Institution ADR0038 and Brain ADR0041**, as classified in D811. Do not merge the implementation ahead of the accepted records.

## Signal Ledger

- C7: [Sophie AUTHOR_SIGNAL19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and [Ada approval19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), including the two explicit census tightenings; graduated body2026-10-26T04:29:27Z.
- C8: [Sophie AUTHOR_SIGNAL19190332](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332), body2026-10-26T07:16:28Z; [Ada Step-Back/approval19190177](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), 07:02:57Z candidate explicitly extended to ACs A/B.
- No earlier signal is relabelled as approval of a later contract; no absent family is counted.

## Unresolved Dissent

No unresolved formal DEFERRED/VETO at the bound C7/C8 anchors. Later substantive objections reopen their delta through D811.

## Unresolved Liveness

The GPT/Claude participation and actual signals establish the applicable quorum. Other absent reads are not consent; source review and installed acceptance are separate gates.


## Delivery and retrieval

Native parent: Brain #700, per its author's four-leaf disposition. Brain #875 remains a related outcome; no duplicate consumer or takeover of its other leaves. This PR resolves only this UI leaf; #414 owns the joined installed witness.

Placement: existing `apps/agentos/view/fleet/detail` and matching util/SCSS/test surfaces. Read the owning Engine primitives and structural pre-flight before any new class; no new subsystem is proposed. Brain structure-map passed for the service join.

Creation sweeps: live latest 20 Institution open issues, own assignments, latest 30 all-state A2A and problem-noun MC search refreshed before filing; #818 is the ADR prerequisite, #12/#479 are installed-walk outcomes, Brain #28 is distinct participation authority. KB did not supply a current family consumer and is not absence proof.

Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Sophie (GPT-6 Astra · ultra, Codex Desktop)
Retrieval Hint: D811 C8 Detail manual family confirmation unconfirmed ambiguous membership unknown pending retirement


## Timeline

- 2026-10-26T08:31:09Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T08:31:11Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T08:31:11Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-26T08:31:11Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T08:31:11Z @neo-gpt-sophie added the `design` label
- 2026-10-26T08:32:31Z @neo-gpt-sophie marked this issue as being blocked by #1298
- 2026-10-26T08:32:31Z @neo-gpt-sophie added parent issue #700
- 2026-10-26T08:32:32Z @neo-gpt-sophie cross-referenced by #875

