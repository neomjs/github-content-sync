---
id: 127
title: A perspective captured while a pane is in its vessel leaves that pane homeless when restored
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T13:17:20Z'
updatedAt: '2026-09-18T16:56:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/127'
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
blockedBy:
  - '[x] 126 The cockpit declares its panes and zones; the hand-built document, the resolver switch and the first-shell mount go'
blocking: []
closedAt: '2026-09-18T16:56:25Z'
---
# A perspective captured while a pane is in its vessel leaves that pane homeless when restored

## Context

Live demo through the Neural Link, 2026-09-12: the inspector (`detail`) was popped out into its own window, then `capture_perspective` (window scope) stored "Gereon-Demo". After the pane had returned home, `restore_perspective('Gereon-Demo')` applied a document whose catalog still lists `detail` (`autoHidden: false`) but whose nodes place it nowhere (`secondary-rail.items = [perspectives, defineAgent, wakeRoutes]`). The inspector vanished from the rail; the vessel chrome stays hidden because nothing is owned or pending; the pane is reachable again only by restoring another perspective.

## The Problem

A window-scope capture snapshots the live dock document as it is. A detached item is absent from the tree by construction (`detachItem` keeps the catalog record and removes the placement), so the capture records the pane's absence as the layout — although the pane is not closed, it is elsewhere, with a home the tear-out owner knows.

## The Architectural Reality

- `VesselContainer.popOutPane` → the engine's `handleDockPopOutAction` → the one `detachItem` commit; `Workspace.applyTearOutOperation` captures the pre-detach placement (`WorkspaceDocument.captureItemPlacement`) for the return.
- `capture_perspective` → the cockpit's perspective store captures `dockModel` (window scope); `restore_perspective` → the fail-closed restore path → `applyDocument`.
- The engine's declarative model (neomjs/neo#18474, neomjs/neo#18492) names the shape: a declared-but-unplaced pane is the detached/closed shape; `Operations.restoreTab({home, index, itemId, tabsNodeId})` returns such a pane to a recorded home; after #126 every cockpit pane has a declared home.

## The Fix

Intake 2026-09-18 settled the fork. `Prescription checked: apps/agentos/view/fleet/cockpit/Controller.mjs#capturePerspective — better owner: neomjs/neo src/dashboard/dock/Workspace.mjs`: the engine holds both the document and the tear-out owner's recorded homes, and its Neural Link capture (`DockService#capturePerspective`) reads the same accessor as the cockpit's verb, so a cockpit-side fold would leave the Neural Link capture recording the hole. The engine half is neomjs/neo#18903 — capture-side: a perspective document with vessel-held panes folded into their recorded homes. **This ticket is blocked on it and on the pin that carries it.**

The restore-side shape is dropped. Since the declared perspectives (#133) placement is what a perspective reveals, so re-seating every unplaced catalog item after a restore would reopen panes a perspective closed on purpose.

What stays here: `Controller#capturePerspective` reads the engine's perspective document instead of `getDockZoneDocument()`, and the Neural Link witness covers capture-while-vesseled → return → restore. The second criterion needs no code — `Container#resolvePane` already answers a stand-in for a vesseled item a restored document places, and the engine's return re-projects the same live pane (`TearOut.mjs`, the "already in the tree" branch) — it needs its witness.

## Acceptance Criteria

- [ ] Capture while `detail` is in its vessel, return the pane, restore the perspective: `detail` is placed in its home tabs node and mounted (`WorkspaceDocument.findContainingTabsId` not null).
- [ ] Capture while vesseled, restore while still vesseled: the pane stays in the vessel; the restored document does not steal it.
- [ ] A unit arm on the cockpit's capture verb reading the engine's perspective document, and one NL witness for the round trip. The fold's own arms live in neomjs/neo#18903.

## Out of Scope

Topology-scope captures (keyed workspaces, the Workstation surface — neomjs/neo#18553); the perspective store's persistence format.

## Related

#126 (declared homes — the restore-side shape depends on it; this lane starts after it); #125 (the vessel layer on the engine's owner); neomjs/neo#18474, neomjs/neo#18492 (declared panes); neomjs/neo#18553 (reset as perspective one, popups standing after a restore).

Live latest-open sweep: the latest 20 open issues of neomjs/neo-agent-institution at 2026-09-12T13:14Z — none equivalent (#103 is the Review preset's FLIP settle, #123 the card's narrow bands). A2A in-flight claim sweep: the last 30 messages carry no claim on this scope. Memory Core rationale sweep: the engine side settled the declared-home return (`restoreTab` from `captureItemPlacement`, the D#18468 fold); nothing decided the capture side. Own-assignment sweep: #126 (related, not the same).

Origin Session ID: fcdd7d71-e7bd-46e8-bd01-5d46ea620205
Retrieval Hint: "perspective capture vesseled pane homeless restore"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fcdd7d71-e7bd-46e8-bd01-5d46ea620205


## Timeline

- 2026-09-12T13:17:20Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-12T13:17:22Z @neo-fable-clio added the `bug` label
- 2026-09-12T13:17:22Z @neo-fable-clio added the `ai` label
- 2026-09-12T14:57:47Z @neo-fable-clio cross-referenced by PR #130
- 2026-09-12T21:01:34Z @neo-fable-clio cross-referenced by #133
- 2026-09-12T21:40:47Z @neo-fable-clio cross-referenced by PR #134
- 2026-09-13T19:10:55Z @neo-fable-clio cross-referenced by #136
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T13:03:35Z @neo-fable-clio cross-referenced by #153
- 2026-09-18T13:40:14Z @neo-fable-clio cross-referenced by #155
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157
- 2026-09-18T15:06:30Z @neo-fable-clio cross-referenced by #18903
- 2026-09-18T15:13:19Z @neo-fable-clio cross-referenced by PR #18904
- 2026-09-18T15:21:07Z @neo-fable-clio cross-referenced by #160
- 2026-09-18T15:30:31Z @neo-fable-clio cross-referenced by #161
- 2026-09-18T15:46:37Z @neo-fable-clio cross-referenced by #163
- 2026-09-18T15:52:45Z @neo-fable-clio cross-referenced by PR #164
- 2026-09-18T16:22:03Z @neo-fable-clio cross-referenced by PR #166
- 2026-09-18T16:56:25Z @tobiu referenced in commit `c100571` - "Merge pull request #166 from neomjs/agent/127-capture-reads-perspective-document

fix(cockpit): a perspective captured while a pane is in its vessel files it in its home (#127)"
- 2026-09-18T16:56:25Z @tobiu closed this issue

