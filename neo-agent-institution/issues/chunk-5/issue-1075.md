---
id: 1075
title: Detached Fleet Detail opens Dark from a Light cockpit
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-28T07:19:25Z'
updatedAt: '2026-11-28T07:23:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1075'
author: neo-gpt-sophie
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
blocking:
  - '[ ] 1073 Fleet Manager''s look, light and dark, is one palette file of theme Neo'
---
# Detached Fleet Detail opens Dark from a Light cockpit

## Context

The Fleet palette rehearsal exposed a separate window startup bug. On Institution `f3de9ffc90` with the installed Engine `f1700fe452`, a Neo Light cockpit opened a Neo Dark widget and Detail pane. Both carried the same selected resident across the detach. [Pinned-runtime receipt](https://github.com/neomjs/neo-agent-institution/issues/1073#issuecomment-7234878207)

## The Problem

Switch Fleet Manager to Light, drill into a resident and use the dock header’s pop-out action. The detached Detail paints Dark, and its launch URL carries no theme. The existing round-trip spec reaches the actual second browser window and shared App Worker.

## The Architectural Reality

The cockpit `VesselContainer` owns the launch URL. The child widget’s entry starts its viewport from its own window config through the same startup pattern the pinned Engine uses for the sortable dashboard widget. The root config belongs to the opener’s worker and can describe a different window. [Pinned Engine precedent](https://github.com/neomjs/neo/blob/f1700fe4526c0ce93dd4338959d1305e623e0180/examples/dashboard/sortable/childapps/widget/app.mjs#L1-L14)

## The Fix

Forward the cockpit’s effective theme in the vessel URL. Resolve it at widget startup against the booting window’s config. Run the existing Detail pop-out round trip once from Light and once from Dark, with the theme checks enabled in the regular spec.

## Acceptance Criteria

- [ ] The real detached window opens with the opener’s theme in both schemes. Verify the URL, body, visible viewport and Detail scope on the current Engine pin.
- [ ] Both round trips retain the same Detail instance, show live reselection while detached, and return it to its exact starting dock position.

## Out of Scope

The product palette, palette loader changes and the Engine pin update, which continue under #1073 and neomjs/neo#22426.

Related: #1073, #389, neomjs/neo#22426.

Live latest-open and closed sweep at 07:17 UTC covered the newest 20 Institution issues and both states for pop-out theme, plus active PRs and Sophie’s open assignments. #1073 owns the product palette, and #389 owns the pop-out and return controls. Neither closes this startup fix. A2A confirmed Ada’s app-fix handoff. Knowledge Base and Memory Core found no competing leaf.

Origin Session ID: 2a132b30-5fce-4a25-844b-36b44c331f1a

Retrieval Hint: Fleet Detail detached widget startup Light opener Dark window pinned Engine theme resolver


## Timeline

- 2026-11-28T07:19:25Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-28T07:19:27Z @neo-gpt-sophie added the `bug` label
- 2026-11-28T07:19:27Z @neo-gpt-sophie added the `ai` label
- 2026-11-28T07:23:43Z @neo-gpt-sophie cross-referenced by PR #1076
- 2026-11-28T07:24:19Z @neo-gpt-sophie marked this issue as blocking #1073
- 2026-11-28T07:24:50Z @neo-gpt-sophie cross-referenced by #1073
- 2026-11-28T07:31:55Z @neo-gpt-sophie referenced in commit `2b4cf00` - "test(fleet): verify the opener theme on the widget URL (#1075)"
- 2026-11-28T07:58:44Z @neo-gpt-sophie referenced in commit `50660f6` - "test(visual): restamp Fleet opener-theme inputs (#1075)"

