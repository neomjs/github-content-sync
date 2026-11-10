---
id: 906
title: Chat leaves the rail until it has a function
state: OPEN
labels:
  - enhancement
  - good first issue
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-10T19:04:37Z'
updatedAt: '2026-11-10T19:05:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/906'
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
milestone: FM v1
---
# Chat leaves the rail until it has a function

## Context

A design sweep of the installed Fleet Manager on 2026-11-10 pressed every door of the rail. The sixth, Chat, opens a view that holds one sentence and no control ([receipt](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6776500148), section `chat`, Institution `86ca206d`).

## The Problem

The view reads: "Chat — prompt an agent → a live widget pane you can dock and pop out. The dockable QT work-area lands here next." The sentence came with the shell's first commit (neomjs/neo#14847, 2026-07-05) and has said "next" since. A first-rank door that leads to a promise costs a new operator one click and a doubt about the other five doors. The views epic proposed in its row line of 2026-10-09 that Chat leaves the rail; no leaf was filed.

## The Architectural Reality

- `apps/agentos/view/Viewport.mjs`: the `agent-placeholder` item with `railHeader('fa-solid fa-comments', '/chat', 'Chat')`.
- `apps/agentos/view/ViewportController.mjs`: the route `'/chat': 'onChatRoute'` and `onChatRoute()`.
- `test/playwright/unit/apps/agentos/ViewportController.spec.mjs` and `apps/agentos/design/default-perspective.html` name the route.
- Design authority: `ROADMAP.md`, "Deferred — explicitly off the v1 path (deferred is not rejected)": "Cockpit-authored Neo apps and conversational UIs (D#18940) … the engine line's showcase, consumed here when a journey asks for it."

## The Fix

Remove the rail item, its body item, and the `/chat` route with its handler. A stored or typed `/chat` hash then behaves as any unknown hash does today. `apps/agentos/design/chat-creation-plan.html` stays: it is the plan, and the door returns with the function.

## Acceptance Criteria

- [ ] AC-1 The rail shows five doors: Home, Fleet, Observatory, System, Accounts.
- [ ] AC-2 Opening `#/chat` lands where an unknown hash lands today; the PR states that behaviour and a spec arm pins it.
- [ ] AC-3 `ViewportController.spec.mjs` and the design pages that list the rail's doors no longer name Chat as a door.
- [ ] AC-4 Post-merge: the next design-sweep receipt for the shell records five doors.

## Out of Scope

The chat feature itself. The Brain's view-key list (`FOCUS_V1_ROUTE_VIEW_KEYS` in `ai/services/graph/focusV1Section.mjs` of neomjs/neo-agent-brain) still names `chat`; that list follows in the Brain after this merges and is not part of this PR.

## Avoided Traps

Rewording the placeholder. A better sentence is still a door without a room.

## Related

#505 (the views epic; its sub-issue list is full, so this leaf links by reference) · neomjs/neo#14847 (origin of the placeholder).

Live latest-open + closed sweep: checked the latest 20 open issues and closed searches ("Catch up", "Chat rail placeholder", "Accounts card read back hooks", "side by side 28rem"; #267 and #84 read as same-surface hits, neither excludes this) at 2026-11-10T19:03Z; no equivalent or prior exclusion found. Backlog band: 47 open, orange; no open ticket owns the rail. A2A claim sweep over the last 40 minutes at 19:03Z: none on this scope. Memory Core rationale sweep on the symptom: no prior decision. Own assignments (#505, #507, #351, #889): #505 is the parent by reference.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 0c3a6369-4269-4b0b-8ad5-0c9441248925

## Timeline

- 2026-11-10T19:04:38Z @neo-fable-clio added the `enhancement` label
- 2026-11-10T19:04:39Z @neo-fable-clio added the `good first issue` label
- 2026-11-10T19:04:39Z @neo-fable-clio added the `agent-os` label
- 2026-11-10T19:04:39Z @neo-fable-clio added the `ai` label
- 2026-11-10T19:04:39Z @neo-fable-clio added the `design` label
- 2026-11-10T19:05:55Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-11-10T19:07:33Z @neo-fable-clio cross-referenced by #505

