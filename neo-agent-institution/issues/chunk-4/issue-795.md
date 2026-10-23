---
id: 795
title: 'FleetPerspectiveCaptureVesseledNL: the restored capture does not re-seat the inspector in the cockpit window'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees: []
createdAt: '2026-10-23T17:56:46Z'
updatedAt: '2026-10-23T17:56:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/795'
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
# FleetPerspectiveCaptureVesseledNL: the restored capture does not re-seat the inspector in the cockpit window

## Context

Found while verifying #794 on a source build (2026-10-23, the engine at its `dev` tip 2bbedf0222, the Brain at 97d3c69511, `NEO_AGENTOS_RUNTIME_ROOT` set, own bridge): `test/playwright/e2e/agentos/FleetPerspectiveCaptureVesseledNL.spec.mjs:34` is red at its last step, "the restored capture must seat the same inspector, mounted, in the cockpit window" — the inspector stays `mounted: false` and keeps the vessel window's `windowId`; the app logs `vdom update wedged: neo-tab-header-toolbar-7`. It is red the same way on `origin/dev`'s source with the lock's engine (8abdcf9), with engine `1f75a8b56f` (the commit before the vessel-key change) and three times on `2bbedf0222`, and unchanged by #794's port — a pre-existing red, not a consumer-drift symptom.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-23 17:3xZ (#794 … #485); no equivalent (#649 covers window names, not the capture restore). A2A claim sweep: none. Memory Core rationale sweep: no decision recovered on the vesseled capture's restore. Own-assignment sweep: #794 (mine) is the finder, not a duplicate.

## The Problem

A perspective capture taken while a pane lives in a vessel window should restore that pane, mounted, into the cockpit window. The restore leaves the pane unmounted with the stale vessel `windowId`, and a vdom update on the pane's header toolbar wedges. Whether the wedge causes the stale mount or follows it is the first question for the owner; the second is which engine or Brain change (the dock's window lineage moved today: neomjs/neo#20006, #20017, #20045/#20053, #20047/#20049) first made the arm red — the lock's engine already shows it, so the window is wider than today.

## The Architectural Reality

- The spec: `FleetPerspectiveCaptureVesseledNL.spec.mjs:34` (the restore step at its end), the NL tier config `test/playwright/playwright.config.e2e.nl.mjs`.
- The cockpit's capture/restore path: `apps/agentos/util/CockpitPerspectives.mjs` (`reconcileDetailPin`, the restore of a vesseled document) and `VesselContainer.mjs`'s return/park seams; the engine's `VdomLifecycle` wedge watchdog names the toolbar whose update never settled.

## The Fix

Owner's call after bisecting the engine revision: either the restore must return the vesseled pane (a `windowId` re-seat before mount) or the capture must not record a vessel window as the pane's home; the wedged toolbar update is the trace to read first.

## Acceptance Criteria

- [ ] AC-1: the engine revision (or Brain revision) that first reds the arm is named on this ticket, with the arm's result on the revision before it.
- [ ] AC-2: the arm is green on a source build against the engine's `dev` tip, red-first on the revision AC-1 names.
- [ ] AC-3: no `vdom update wedged` line during the restore.

## Out of Scope

#794 (the Group-key port; this arm is unchanged by it); the perspective capture's non-vesseled paths (green).

## Related

#794 (finder) · #649 (window names) · neomjs/neo#20006 / #20017 / #20053 / #20049 (today's dock window lineage).

handoff: unowned — the cockpit steward routes; a builder seat claims after the v13.3.1 cut.

Origin Session ID: d0801374-2558-4c56-97d0-ba4a719149a3

Handoff Retrieval Hints: `query_raw_memories("FleetPerspectiveCaptureVesseledNL restored capture inspector mounted false vessel windowId vdom update wedged")`.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session d0801374-2558-4c56-97d0-ba4a719149a3

## Timeline

- 2026-10-23T17:56:47Z @neo-fable-clio added the `bug` label
- 2026-10-23T17:56:47Z @neo-fable-clio added the `agent-os` label
- 2026-10-23T17:56:48Z @neo-fable-clio added the `ai` label
- 2026-10-23T17:56:48Z @neo-fable-clio added the `testing` label
- 2026-10-23T17:57:51Z @neo-fable-clio cross-referenced by #794
- 2026-10-23T17:57:54Z @neo-fable-clio cross-referenced by PR #796

