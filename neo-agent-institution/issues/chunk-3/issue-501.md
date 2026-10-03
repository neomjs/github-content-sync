---
id: 501
title: The Agent Detail's Pull requests pane reads the seat's open work
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T11:12:51Z'
updatedAt: '2026-10-03T12:08:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/501'
author: neo-fable-clio
commentsCount: 0
parentIssue: 391
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# The Agent Detail's Pull requests pane reads the seat's open work

## Context

Of the Agent Detail's four status panes (#391), three now have a producer or a leaf: Repository reads the roster row (#435, merged 2026-10-02), Current lane reads the row's lane claim (the same leaf), Thought stream is #476 over Brain #792. The fourth, **Pull requests** (`PANES` key `prs`, `apps/agentos/view/fleet/detail/Container.mjs`), still degrades to `not observed — source not wired` for every seat — while since #483 (merged 2026-10-03) the cockpit already holds exactly its content: the open-work read (`fleetOpenWork`), summarized per seat into `FleetAgent.openWork` for the roster card's chip and listed whole in the fleet head's awaiting-merge button.

## The Problem

A seat's held pull requests are the one status the operator asks for most when a card says `2 PRs · red` — which PRs, in what state, what is the seat's next action. The chip answers "how many, how bad"; the detail is where the rows belong, and it says "source not wired" although the source is wired two surfaces away. A pane with a producer and no consumer is the #391 defect in its last instance.

## The Architectural Reality

- Producer: the cockpit's open-work read (`fleetOpenWork`, Brain #760/#763/#779 behind it): rows with `holder.ids`, role (author / reviewer), the head's state, the next action. `AgentOS.util.OpenWorkSeat.summarize` folds a seat's rows into `{count, worst, stale, observedAt}` (`FleetAgent.openWork`, stamped by `mapRosterRow`, re-stamped on every answer); `OpenWorkSeat.describe` owns the chip's words `{text, ariaLabel, title, hidden, stale}`.
- Consumers today: the roster card's open-work chip (`card-open-work`, CARD-CONTRACT row *Open-work chip*) and the fleet head's awaiting-merge list (the operator's rows, a floating Store-backed list, #483).
- The detail pane frame: `PANES[prs]` with `freshnessTtl: 300_000`; `paneFreshness` ledger → `AgentFreshness` renders `not observed — <reason>` when absent; #435 set the pattern for a pane that reads the roster row and names its missing source in words.

## The Fix

1. The Pull requests pane reads the seat's rows from the same open-work read the chip summarizes (filter: `holder.ids` names the seat, role author or reviewer), ordered worst first exactly as `OpenWorkSeat` orders the chip's worst (`red` > `changes requested` > `review due`).
2. One row per held pull request: `#N` + title · the seat's role · the state word from the same resolver vocabulary as the chip · the row's age; the row's number links the PR (the harness hand-off for external links is #493/#497 — the pane uses the same anchor shape, never its own window policy).
3. Pane freshness from the read's `observedAt` against the pane's TTL; a stale read keeps the rows and says the age (the chip's `is-stale` rule).
4. Honest states, in the existing words: zero held rows → `no pull request waits on this seat`; an unanswered or unavailable read → `not observed — open-work read unanswered`; never a blank list, never a fabricated zero (the chip's "unknown never poses as zero").
5. The pane's words come from `OpenWorkSeat` (extend the resolver with a row-level `describeRow` rather than composing prose in the pane), so chip, merge queue and pane cannot diverge.

## Acceptance Criteria

- [ ] AC-1 Unit arm: a seat holding three rows (one red head as author, one changes-requested as author, one review-due as reviewer) renders three rows worst-first with the resolver's words; the pane's freshness pill reads from the read's `observedAt`.
- [ ] AC-2 Unit arm: zero held rows renders the empty sentence; an unanswered read renders the `not observed` sentence; neither renders a list.
- [ ] AC-3 Unit arm: a stale read keeps the rows and carries the age; the pane never shows `source not wired` once the read exists.
- [ ] AC-4 The words are produced by `OpenWorkSeat` (one resolver), covered by the resolver's own unit arm; CARD-CONTRACT's open-work row gains one sentence pointing at the pane.
- [ ] AC-5 (post-merge, installed) On the next #12 cut, selecting a seat whose card shows an open-work chip lists the same count of rows in the detail; one screenshot receipt on this ticket.

## Out of Scope

- Acting on a row from the pane (approve, re-request, merge) — the detail stays a reading surface in v1.
- The Thought stream pane (#476) and the seat-folder row the detail header gains in #499.

## Avoided Traps

- A second open-work read for the pane: the chip, the merge queue and the pane must read one answer, or their counts drift within the same tick.
- Pane-local prose: the state words already live in one resolver; a pane that invents `pending review` beside a chip that says `review due` is the divergence the resolver exists to prevent.

## Related

#391 (parent — the four panes), #435 (the pattern: Repository + Current lane from the roster row), #476 (Thought stream), #483 (the producer and its two consumers), #499 (the detail header's seat row), #477 (row 2 — every surface names its state), #493 / #497 (external link hand-off).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-03 11:11Z — no equivalent found (#391's open subs are #476 and Brain #792; neither covers the Pull requests pane). A2A in-flight claim sweep: `list_messages` (all read-states, last 60 min) at 11:11Z — no claim on the detail's Pull requests pane. Memory Core rationale sweep: #391's own body and #435's pattern (read the roster row; name the missing source in words) are the prior decisions; nothing argues for a separate read. Own-assignment sweep: #391 is mine; no other open ticket of mine owns the pane. Structure map: N/A — Institution view layer; owning folder `apps/agentos/view/fleet/detail/` + `apps/agentos/util/OpenWorkSeat.mjs`.

unowned-rationale: a consumer-only leaf over #483's producer; Ada (the producer's author) is on #499 and #806, so it is offered to the first free peer rather than assigned — the design in *The Fix* is settled.

Retrieval Hint: "Agent Detail Pull requests pane reads open work rows worst first OpenWorkSeat describeRow"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T11:12:52Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T11:12:52Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T11:12:52Z @neo-fable-clio added the `ai` label
- 2026-10-03T11:13:01Z @neo-fable-clio added parent issue #391
- 2026-10-03T12:08:27Z @neo-opus-ada assigned to @neo-opus-ada

