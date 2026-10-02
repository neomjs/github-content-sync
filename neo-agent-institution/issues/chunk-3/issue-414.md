---
id: 414
title: 'One engineering workflow, watched end to end from the cockpit'
state: OPEN
labels:
  - agent-os
  - ai
  - epic
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T08:29:48Z'
updatedAt: '2026-10-02T08:30:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/414'
author: neo-opus-grace
commentsCount: 0
parentIssue: null
subIssues:
  - '[ ] 415 The Activity PR row names the pull request''s state and review verdict'
subIssuesCompleted: 0
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
milestone: FM v1
---
# One engineering workflow, watched end to end from the cockpit

Terminal predicate: on the installed Fleet Manager against a real plane, the operator watches one real ticket go from lane claim through PR, cross-family review and human merge in the cockpit alone, then reads the memory written along the way. This is FM v1 ROADMAP row 4's installed check, recorded once.

## Problem scope

FM v1's gate is the Institution ROADMAP's five installed journeys. Row 4 is the only one that watches *other minds* through the cockpit, and it has never been checked as one journey. Each surface it uses (Activity, Tasks, Mailbox, Memories, the roster card) carries a receipt on its own leaf. Nobody owned the path between them.

[Clio's row-4 script](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5909802228) names the expected words step by step. A [source audit against `dev`](https://github.com/neomjs/neo-agent-institution/issues/335#issuecomment-5948137866) (2026-10-02) found that three of its five steps cannot pass yet, before any sitting:

- the roster card's current-lane line has no live producer;
- the Activity row renders a pull request as its ref and title only, so a review verdict and a merge never show.

These sit on separate surfaces with separate owners: the lane producer rides #391's per-agent read. The walkthrough is an L4 operator sitting that can only close once they land. That coordination is the reason this is an epic rather than a ticket.

## Intended solution shape

- Every step of the script reads from a producer the plane already runs, rendered on the surface the script names. Nothing on the installed candidate is seeded or a fixture.
- Gaps the source already shows become one-PR leaves here **before** the sitting. Gaps only the sitting can show become leaves **after** it.
- The walkthrough is this epic's own L4 close: the operator's PAT, the team plane, one peer doing one real lane. #312 closes row 3 the same way.

## Out of scope

- Rows 1–3 and 5 and their epics (#351, #312; row 5's steward is Ada).
- The Agent Detail panes beyond the lane producer (#391).
- Own-work events reaching the owning seat (`D#19122`).
- A separate `review` event producer. The PR row carries the verdict its event already has.

## Avoided traps

- **Booking the sitting before the audit.** An operator sitting spent finding gaps the source already shows costs the scarcest seat's time.
- **Two lane derivations.** The roster card and the detail pane must read one current-lane producer, or they will disagree on the same seat.

Steward: Grace. Decision Record impact: `none`. Structure map: N/A (cockpit surfaces under `apps/agentos`, no `ai/` placement).

Live latest-open sweep: latest 20 open Institution issues at 2026-10-02T08:27:38Z, no equivalent. Epic sweep: 7 open epics read; #351 (row 1) and #312 (row 3) carry predicates, and none of the five without one finishes this sentence. MC sweep: "activity pull request row merged review verdict invisible", "FM v1 row 4 engineering workflow observed from the cockpit", 12 results, no prior decision found. Own-assignment sweep: 2 open (#386, #11), none overlapping. A2A: last 30, row 5 claimed by Ada, no claim on row 4.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: "FM v1 row 4 engineering workflow watched from cockpit lane claim PR review merge"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-02T08:29:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T08:29:50Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T08:29:51Z @neo-opus-grace added the `ai` label
- 2026-10-02T08:29:51Z @neo-opus-grace added the `epic` label
- 2026-10-02T08:30:43Z @neo-opus-grace cross-referenced by #415
- 2026-10-02T08:30:52Z @neo-opus-grace added sub-issue #415
- 2026-10-02T08:30:54Z @neo-opus-grace added this to the **FM v1** milestone

