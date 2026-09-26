---
id: 240
title: 'The tasks pane ships no sample rows: cold and empty sections are its own'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-09-26T09:24:57Z'
updatedAt: '2026-09-26T09:24:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/240'
author: neo-fable-clio
commentsCount: 0
parentIssue: 237
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
# The tasks pane ships no sample rows: cold and empty sections are its own

## Context

Third leaf of #237. The tasks pane "teaches its shape before any bridge answered — exactly like the static roster" (`apps/agentos/view/fleet/tasks/Container.mjs` ~22–43): `SAMPLE_ROWS` per section and a `SAMPLE_SCHEDULER` render whenever the pane is cold, with a `sample` pill. The operator's ruling applies: either there is real data, or there is not.

## The Problem

A cold tasks pane shows invented queued, starved and backup rows with a `sample` source pill. It is labelled, but it is still a picture of work that does not exist — the same claim-dressed-as-state the roster made.

## The Architectural Reality

- `tasks/Container.mjs`: `SAMPLE_ROWS`, `SAMPLE_SCHEDULER`, the `cold ? 'sample' : wired ? 'live' : 'unavailable'` pill (~245), the section rows composed from `SAMPLE_ROWS` when cold (~279), the hoisted source logic (~281–294); `tasks/List.mjs` renders `record.sample ? 'sample' : …` (~288); `model/FleetTask.mjs` carries a `sample` field (~110).
- The visual goldens `tasks-pane-240*` and the band arms (301 · 400 · 649) measure the dense sample rows (`FleetCockpitVisual.spec.mjs` ~588–650) — they need rows to measure, so they land a fixture through a driver (the first leaf's shape, extended with tasks rows).

## The Fix

1. Remove `SAMPLE_ROWS`, `SAMPLE_SCHEDULER` and the `sample` field/pill; cold = each section empty with a "not answered yet" line; a live empty answer = "no tasks" per section; the scheduler line renders only from a wired snapshot.
2. The tasks fixture joins the test fixtures (rows + scheduler) with a driver landing; the width-band arms and the 240 goldens land it first; cold goldens show the empty sections.

## Acceptance Criteria

- [ ] AC-1 `grep -n "SAMPLE_\|sample" apps/agentos/view/fleet/tasks apps/agentos/model/FleetTask.mjs` returns nothing; the pane cold renders empty sections with the cold line (unit spec).
- [ ] AC-2 A live empty snapshot renders "no tasks" per section and the live pill; the scheduler line only from a wired snapshot (unit spec).
- [ ] AC-3 The visual width-band arms and `tasks-pane-240*` land the tasks fixture through the driver; the cold capture shows the empty sections; stamp re-issued; suite green.

## Out of Scope

The tasks read itself (Brain #322 shape); the roster and activity (the second leaf).

## Related

#237 (parent), the first and second leaves, #11.

unowned-rationale: the third leaf lands after the second; mine unless a peer claims it once the second is open.

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:22:32Z — no equivalent. A2A sweep: none. Memory Core sweep: none. Own-assignment sweep: #237 only. Structure map: N/A.

Origin Session ID: 26b775fe-f8d9-4258-809c-09d9e5ef8ed1
Retrieval Hint: `query_raw_memories("tasks pane sample rows retired cold empty sections fixture driver")`

## Timeline

- 2026-09-26T09:24:59Z @neo-fable-clio added the `enhancement` label
- 2026-09-26T09:24:59Z @neo-fable-clio added the `agent-os` label
- 2026-09-26T09:24:59Z @neo-fable-clio added the `ai` label
- 2026-09-26T09:24:59Z @neo-fable-clio added the `design` label
- 2026-09-26T09:25:18Z @neo-fable-clio added parent issue #237
- 2026-09-26T12:25:38Z @neo-fable-clio cross-referenced by #239
- 2026-09-26T12:44:06Z @neo-fable-clio cross-referenced by PR #254

