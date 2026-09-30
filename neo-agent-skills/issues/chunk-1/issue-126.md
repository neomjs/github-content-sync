---
id: 126
title: 'epic-create names an epic''s size: around 25 subs, then a successor epic'
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-09-30T12:42:40Z'
updatedAt: '2026-09-30T12:42:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/126'
author: neo-fable-clio
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
blocking: []
---
# epic-create names an epic's size: around 25 subs, then a successor epic

## Context

Operator ruling, 2026-09-30, on neomjs/neo-agent-institution#10 (a cockpit epic that had grown to 100 native sub-issues, 95 closed): *"an epic with 95 subs is no longer reasonable. an epic should have around 25 subs. no one can check if an epic is done, in case there are way too many subs."* The mechanical symptom arrived the same hour: GitHub refused the 101st sub-issue link (its cap is 100), and the epic had to be closed through a resolution matrix built over outcome clusters because a per-sub matrix was unbuildable.

## The Problem

`epic-create-workflow.md` says when an epic is warranted (leaves coordinate toward one shared outcome) and how subs are linked (natively, never listed in the body), but nothing says how many leaves an epic can carry before it stops being one outcome. `ticket-create` §10 links a new sub with `update_issue_relationship` without counting the parent's subs. So epics accrete until the count is a wall: `epic-resolution`'s matrix is per parent AC, but at 95 subs nobody builds it and the epic can neither close nor tell a reader what remains.

## The Architectural Reality

- `epic-create-workflow.md` (Procedure 1: the epic test; the body carries problem-scope + intended-solution, subs are linked) — the right home for a sizing rule, one paragraph.
- `ticket-create-workflow.md` §10 (after creation: `update_issue_relationship(parent_id, child_id, 'SUB_ISSUE')`) — the right hook for a count-before-link line.
- `epic-resolution-workflow.md` §3 (the matrix) — the consumer that the size rule protects; the closeout of #10 shows the fallback (rows = the body's intended-solution bullets, subs counted per cluster).
- GitHub's native sub-issue cap is 100 per parent (REST: `POST /issues/{n}/sub_issues` fails past it).

## The Fix

1. `epic-create-workflow.md`: a **Size** paragraph under the epic test — *around 25 subs is the working size; an epic is a checkable outcome, and past that count the leaves no longer share one. Count the parent's subs before filing under it (`gh api repos/<r>/issues/N/sub_issues --paginate --jq length`); past ~25, file a successor epic with its own terminal predicate and link the new leaf there. A saturated epic gets the closeout treatment: cluster its open subs into outcomes, file the successors, re-parent, resolve the old one.* GitHub's cap of 100 named as the hard wall.
2. `ticket-create-workflow.md` §10: one line at the `update_issue_relationship` step — count first; past ~25, link to a successor.
3. Substrate size discipline: the corpus net-growth arm requires the `[skill-growth-justified: …]` trailer above 250 bytes of growth — the justification is this ruling and the #10 closeout.

## Acceptance Criteria

- [ ] AC-1 `epic-create-workflow.md` carries the Size paragraph (the ~25 working size, the count command, the successor-epic path, the cap of 100).
- [ ] AC-2 `ticket-create-workflow.md` §10 counts before linking and names the successor path in one line.
- [ ] AC-3 The parity/materialize checks pass (`--update-parity` if a leaf changes); growth justified by trailer.

## Out of Scope

A mechanical guard in the MCP `update_issue_relationship` tool (a Brain change; worth its own leaf if the discipline line proves insufficient); re-splitting existing large epics (per-epic closeouts, as done for neomjs/neo-agent-institution#10).

## Related

neomjs/neo-agent-institution#10 (the closeout that triggered this) · neomjs/neo-agent-institution#349 (the refused link)
Live latest-open sweep: checked the latest 15 open issues at 2026-09-30T12:42Z; no equivalent (#62 sweeps closed state, #86 the AC table — adjacent, not this). A2A claim sweep (last 30, all read-states): no claim on this scope. Memory Core sweep: the operator's ruling is the origin. Own-assignment sweep: none in this repo.
Decision Record impact: none.
unowned-rationale: a two-paragraph substrate edit any seat can land; the design seat recorded the ruling and holds the wording if asked.

Origin Session ID: 4a2cca3d-9951-4e9a-b577-2a3374a22045
Retrieval Hint: "epic size around 25 subs successor epic GitHub sub-issue cap 100"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

## Timeline

- 2026-09-30T12:42:41Z @neo-fable-clio added the `enhancement` label
- 2026-09-30T12:42:42Z @neo-fable-clio added the `ai` label

