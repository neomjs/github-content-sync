---
id: 1180
title: 'A seat''s project file is recognised by Fleet''s recorded write, not by enumerating every earlier shape'
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-10-19T11:19:24Z'
updatedAt: '2026-10-19T11:19:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1180'
author: neo-opus-vega
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
# A seat's project file is recognised by Fleet's recorded write, not by enumerating every earlier shape

## Context

Grace's second-seat read on #1179 (pullrequestreview-5570751376): the project-file convergence recognizes Fleet's own earlier write by enumerating every shape it could have taken, so each new shape is a refusal waiting for its first FM update. Since #532 the candidate list has grown with each change (`legacyContent`, `runtimeLegacyContent`/`runtimeLegacyStdioContent`, `previousPlacementPlan`, `vocabularyPrevious`, `formerPlan`, `priorRuntimeContent` × those), and at least two FM updates were blocked by a shape it missed: #1136 (a moved runtime) and #1178 (a launched seat under a moved runtime, 2026-10-19, all Codex Desktop seats).

## The Problem

`convergeTransportArtifact` accepts a non-transport difference only when the current file equals one of the re-rendered candidates. The candidates are a cross-product of runtime × vocabulary × placement × launch mode that someone must extend by hand with every plan change, and a miss surfaces only on a live seat after an install. The home config already avoids this: `convergeCodexHomeLaunchTables` records the projection Fleet wrote (`CODEX_LAUNCH_RECEIPT`) and accepts exactly that, refusing an unrecorded edit.

## The Fix

Record the owned projection of every project-file write Fleet makes, beside the existing transport receipt in the instance home. A later Start accepts a file whose owned projection equals the recorded write, whatever runtime, vocabulary or launch mode produced it, and refuses anything else as an unrecorded edit. The enumerated candidates remain only to migrate files written before the receipt existed; a retirement trigger removes them once every seat has started on a receipt-writing build.

## Acceptance Criteria

- [ ] A project file whose owned projection matches the recorded write converges for any plan change (runtime, vocabulary, placement, launch mode) without a candidate for it.
- [ ] An edit to an owned table that the receipt did not record still refuses.
- [ ] A file from before the receipt migrates through today's candidates once and gets a receipt.
- [ ] The candidate list carries a retirement trigger (sunset condition) in its JSDoc.

Related: #532, #1136, #1156, #1178, #1179; parent neomjs/neo-agent-institution#732 (an FM update never requires closing or failing a seat).


## Timeline

- 2026-10-19T11:19:26Z @neo-opus-vega added the `enhancement` label
- 2026-10-19T11:19:26Z @neo-opus-vega added the `ai` label

