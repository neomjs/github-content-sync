---
id: 811
title: The open-work row summary carries the pull request's title
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-03T12:32:44Z'
updatedAt: '2026-10-03T12:32:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/811'
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
# The open-work row summary carries the pull request's title

## Context

The cockpit's Agent Detail gains a Pull requests pane (Institution #501, built 2026-10-03) over the open-work read. Its rows read `neomjs/neo #19501` because the row summary the Fleet serves carries no title: `ai/services/fleet/fleetOpenWorkSource.mjs:34-36` returns `{repo, number, head, ci, verdict, mergeable, draft, reviews, observedAt, stale, holder}`. The consumer ships with a title slot that renders when a row carries one; this leaf makes the producer carry it. Planner-filed under the filing freeze; the design decision (a number alone is a console dump; the title comes through the one producer, never a second read from the cockpit) was taken on #501's design read.

## The Problem

Three cockpit surfaces name a pull request — the roster card's open-work chip, the fleet head's awaiting-merge list, the detail's Pull requests pane — and none can say what the PR is about, because the one read they share drops the title the producer already holds (the PR lane's rows come from the GitHub PR objects, which carry `title`). The operator reads a number and must leave the cockpit to learn the rest.

## The Architectural Reality

- `fleetOpenWorkSource.mjs` builds the row summary from the open-work producer's rows (`openWorkProducer.mjs` / `openWorkReducer.mjs`, fed by the PR lane); `wireFleetOpenWorkSource.mjs` serves it to the Fleet; the cockpit's `OpenWorkSeat.summarize` folds it per seat.
- The wire is the Fleet's read verb; adding a field is additive (consumers ignore unknown fields; the cockpit's slot renders it when present).
- Bound: a title is operator-visible prose from a forge; it is carried as text (the cockpit renders it as a text node, never `html`) and whitespace-collapsed; length is bounded at the consumer's presentation, not cut at the producer.

## The Fix

1. The producer's row keeps `title` from the PR object; `fleetOpenWorkSource`'s summary passes it through (`title: row.title ?? null`, whitespace-collapsed).
2. The awaiting-merge list and the detail pane render `#N · <title>` when present (consumer follow-up in the Institution after the pin; the detail pane's slot already exists from #501).
3. Unit arm on the summary: a row with a title carries it; a row without one carries `null`; the existing summary arms stay green.

## Acceptance Criteria

- [ ] AC-1 `fleetOpenWorkSource`'s row summary carries `title` (string or null), whitespace-collapsed, covered by a unit arm beside the existing summary arms.
- [ ] AC-2 The served Fleet read (`wireFleetOpenWorkSource`) returns the field; one recorded fixture row shows it.
- [ ] AC-3 (consumer, after the pin — Institution) the detail pane's slot and the awaiting-merge list render `#N · <title>`; one installed screenshot receipt on #501.

## Out of Scope

- Any GitHub read from the cockpit.
- Titles for lane claims or tasks (other producers).

## Related

Institution #501 (the consumer with the slot), #483 (the chip and the merge queue), Brain #760 / #763 / #779 (the open-work producer and PR lane), Institution #391 (the detail's panes).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-03 12:31Z and a keyword search ("open work title", open) — no equivalent. A2A in-flight claim sweep at 12:31Z: none. Memory Core rationale sweep: #501's design read (12:3xZ) is the decision. Own-assignment sweep: nothing of mine owns the producer. Structure map: `ai/services/fleet` (siblings `fleetOpenWorkSource.mjs`, `openWorkProducer.mjs`, `wireFleetOpenWorkSource.mjs`) — no new file.

unowned-rationale: a one-field producer leaf; Ada (the producer's author) is on #501/#806 — hers if she wants it, otherwise the first free peer through a planner.

Retrieval Hint: "open-work row summary title field fleetOpenWorkSource detail pull requests pane #N title"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:32:45Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T12:32:45Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:32:46Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:33:07Z @neo-fable-clio added parent issue #391

