---
id: 308
title: Define System around containers and real maintenance progress
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T10:09:51Z'
updatedAt: '2026-09-29T11:45:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/308'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 13
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-29T11:45:49Z'
---
# Define System around containers and real maintenance progress

## Context

Operator product direction, 2026-09-28: focus first on our own FM setup and real team; outbound onboarding for other operators and teams is a separate journey. The System container inventory is useful. Its Scheduler must represent the **real cloud-plane orchestrator's heavy-maintenance tasks**, especially their order and progress. Strong visuals should make that work understandable.

The installed screenshot shows clipped service content and raw detector vocabulary such as “restart churn”. The current Tasks tab shows maintenance observations; its existence does not satisfy the intended scheduler experience.

## The Problem

The content and user journey need definition before another styling pass. This is a **design/specification deliverable** using the existing System artifact, not an implementation ticket for a guessed API.

## The Architectural Reality

- `apps/agentos/design/institution-system-view.html` is the existing reference.
- `view/system/Container.mjs` and `List.mjs` present provider-owned deployment state; `view/fleet/tasks/` presents the task feed.
- Brain `fleetTasksSource.mjs#orderSection` currently computes display order, not execution order. Some sources expose progress (KB ingestion and REM backlog); universal task progress is not established.
- Existing #21 delivered the bounded System reader; #240 owns removal of task sample data; #247 owns shared pane chrome. None defines this newly clarified product outcome.

## The Fix

Update the existing design artifact in one reviewable change, in this order: purpose → operator and peer questions → content/actions → source coverage → interaction → visual treatment. Define the container overview and scheduler's relationship inside System. Use captured real data for the acceptance walkthrough and label any sketch fixtures explicitly.

The design should make running work, current phase/progress, upcoming work and waiting reasons legible. Distinguish measured progress, backlog and unknown progress. Keep technical diagnosis inspectable without letting raw fields determine the overview.

## Contract Ledger

| Design surface | Authority | Deliverable | Evidence |
|---|---|---|---|
| Containers and maintenance schedule | Operator direction; existing source-owned state | Product brief and rendered System design | AC-1–3 |
| Needed data | Actual scheduler/read contracts | Coverage table: available, missing, uncertain; owner for each gap | AC-2 |

## Acceptance Criteria

- [ ] AC-1 The brief identifies human and peer outcomes, and explains why each primary piece of content is present.
- [ ] AC-2 Trace scheduler ordering, phase and progress to their actual producer. Name any missing contract explicitly; do not relabel display sorting as execution order.
- [ ] AC-3 The rendered design supports a walkthrough over our real workload: identify a container problem, inspect running maintenance and its progress, understand what follows or waits. Validate readable layout and purposeful motion in both themes and reduced motion.
- [ ] AC-4 Record the operator/peer design decisions in the artifact and map bounded implementation gaps to existing work or justified successors. Runtime implementation does not start under this design-only issue.

## Out of Scope

A new scheduler, restart/pause/reorder commands, new Brain APIs, deployment changes, or graph implementation.

## Avoided Traps

Styling the current field dump before selecting content; invented percentages/ETAs; hiding clipping; treating an animated prototype as live runtime proof.

Decision Record impact: none proposed; implementation/API choices remain open.
Structure map: N/A — existing Institution HTML design artifact, no new module.
Related: #13 (parent), #21, #240, #247.

unowned-rationale: preserved for peer self-selection after the graph-first milestone; no parallel implementation claim.

Freshness sweep: latest 20 open Institution issues, relevant closed #20/#21 and recent all-state A2A checked 2026-09-28; no equivalent. Own Institution assignments: none open after #302 merged; its installed validation remains Emmy's. MC `a48bee58-8e15-4923-8e4d-533bcd03d0a9` preserves the operator clarification.

Origin Session ID: 23b22a41-52ac-4e6c-8d80-23d54054c48c
Retrieval Hint: `System real cloud-plane heavy-maintenance ordering progress product design`
Authored by Emmy (GPT-6 Astra, Codex).

## Timeline

- 2026-09-28T10:09:53Z @neo-gpt-emmy added the `enhancement` label
- 2026-09-28T10:09:53Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-28T10:09:53Z @neo-gpt-emmy added the `ai` label
- 2026-09-28T10:09:53Z @neo-gpt-emmy added the `design` label
- 2026-09-28T10:09:56Z @neo-gpt-emmy added parent issue #13
- 2026-09-28T10:11:21Z @neo-gpt-emmy cross-referenced by #247
- 2026-09-28T10:57:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T11:41:48Z @neo-opus-vega cross-referenced by #310
- 2026-09-28T15:06:09Z @neo-opus-vega cross-referenced by PR #317
- 2026-09-28T16:29:46Z @tobiu referenced in commit `735e919` - "docs(agentos): the System design marks its schedule, times and progress by their sources (#308)

Addresses review 5341523452.
- The Next column says it is the proposed contract, and each job names
  the picker rank it competes on. Section 4 lists all five ranks.
- Frame 1's head marks it as an illustration composed from reads
  between 13:00 and 14:56. The stale and missing states are separate
  frames.
- Every container row opens its detail in place, and the Maintenance
  heading links to the timeline. The separate opened-row frame is gone.
- The REM bar is hatched, because inside a chunk nothing counts. The
  undigested count is a bounded sample of the latest 2,000 sessions.
  Frame 2 shows the three encodings: measured, not measured and
  sampled.
- Both animations run on the theme's --fm-motion-pulse."
- 2026-09-28T17:28:19Z @tobiu referenced in commit `91e3c50` - "docs(agentos): the System design's Next lists candidates, not an order (#308)

The picker's choice between two staleness jobs needs the waiter ledger
and each task's last run, and the plane publishes neither. The Next
column now groups those jobs unranked beside the rule that decides."
- 2026-09-28T17:28:36Z @neo-opus-vega referenced in commit `b6ac0dd` - "docs(agentos): renders of the System design at 91e3c50 (#308)"
- 2026-09-28T17:45:50Z @neo-opus-vega cross-referenced by PR #318
- 2026-09-29T11:45:49Z @tobiu referenced in commit `17b9052` - "Merge pull request #317 from neomjs/vega/308-system-definition

docs(agentos): System design v4 shows the plane diagnosing and healing itself (#308)"
- 2026-09-29T11:45:49Z @tobiu closed this issue

