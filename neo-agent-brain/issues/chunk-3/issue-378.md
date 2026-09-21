---
id: 378
title: 'A Round 2 cannot disposition a micro-form review: its Findings checklist is not read as the prior round''s Required Actions'
state: CLOSED
labels:
  - bug
assignees:
  - neo-opus-ada
createdAt: '2026-09-19T15:52:55Z'
updatedAt: '2026-09-19T16:24:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/378'
author: neo-opus-ada
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
closedAt: '2026-09-19T16:24:28Z'
---
# A Round 2 cannot disposition a micro-form review: its Findings checklist is not read as the prior round's Required Actions

## Context

The validator half of neomjs/neo-agent-skills#64. That ticket also covers the guide and template half. `manage_pr_review`'s Round-2 relation check reads the prior `CHANGES_REQUESTED` review's actions with `extractRequiredActions()` in `ai/services/github-workflow/PullRequestService.mjs`. That function only looks under a heading containing "Required Actions". The micro form (`pr-review-micro-review-template.md`) carries its actions as a `- [ ]` checklist under a bold `**Findings:**` label, so a micro `CHANGES_REQUESTED` yields no actions. Its Round 2 is then refused with "lists no Required Actions … Use the follow-up template".

## The Problem

Four receipts: neomjs/neo#18550 (2026-09-09), #18811 and #18819 (09-16/17, @neo-opus-grace), and **#18974 today**. On #18974, @neo-gpt-emmy's Round-2 approval over her own micro request-changes was rejected because the tool "did not extract a Required Actions section from my original Micro-Review's Findings checklist". The guide's §6.4 tells a reviewer to use the micro form on a mechanical PR, and that choice silently removes the ordinary Round 2.

## The Fix

`extractRequiredActions()` also reads the micro form. When a body has no Required Actions heading, the actions are the `- [ ]` items under its `**Findings:**` label, up to the next bold label or heading. Everything after that is unchanged: the verbatim rule, the row count, the state matrix and every refusal.

## Acceptance Criteria

- [ ] A Round 2 that quotes a micro review's `- [ ]` Findings verbatim is accepted.
- [ ] The relation still refuses the same body with a reworded or invented action, and with a missing row. These arms are the "malformed is still refused" half of Skills #64's AC-3.
- [ ] A micro review whose Findings are `None` still lists no actions, and its Round 2 is still refused.
- [ ] Every existing Round-2 arm over a full-form prior passes unchanged (Skills #64 AC-4).

## Out of Scope

- The guide/template wording that names this shape for cycle 2 (Skills #64 AC-1/AC-2).
- A micro form submitted *as* a follow-up cycle, refused for missing full-form anchors. That is Skills #64's first observation, a different path.

## Related

neomjs/neo-agent-skills#64 (parent) · neomjs/neo#18974 (today's receipt)

Live latest-open sweep: the latest 20 open issues in this repository at 2026-09-19T15:56Z, plus all-state searches for `Round 2 micro` and `Findings extractRequiredActions`. The only hit is #34 (Round 2 terminal across every action-demand channel), which is related and not equivalent.

Origin Session ID: 6ecb7b5f-dc26-48a3-8e49-7232159377c1
Retrieval Hint: "Round 2 micro review Findings checklist extractRequiredActions manage_pr_review disposition"


## Timeline

- 2026-09-19T15:52:55Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-19T15:52:56Z @neo-opus-ada added the `bug` label
- 2026-09-19T15:57:13Z @neo-opus-ada cross-referenced by PR #379
- 2026-09-19T16:24:28Z @tobiu referenced in commit `77ee09e` - "Merge pull request #379 from neomjs/ada/378-round2-micro

fix(github-workflow): a Round 2 can disposition a micro-form review's Findings checklist (#378)"
- 2026-09-19T16:24:29Z @tobiu closed this issue
- 2026-09-19T16:31:53Z @neo-opus-ada cross-referenced by #380

