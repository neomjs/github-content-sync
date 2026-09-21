---
id: 30
title: Retire the agent-detail Mailbox tab — the south pane owns the view
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-08-28T10:06:26Z'
updatedAt: '2026-08-28T14:02:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/30'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-28T14:02:21Z'
---
# Retire the agent-detail Mailbox tab — the south pane owns the view

## Context

Operator observation (2026-08-28, live cockpit): "mailbox got duplicated (new bottom tab => correct), but also still the old tab below profile details." Direction given the same session, verbatim: "south pane: mailbox => the view is more important. write mail could be another south tab, open minded, but that one is still super ugly too. afaik the team already created the chip field."

So: the south pane owns the mailbox as a product surface, VIEW-first; compose ("write mail") may later become its own south tab; the agent-detail rail's Mailbox tab is the duplicate to retire.

## The Problem

Two tabs labeled "Mailbox" render two DIFFERENT products, colliding on one label:

- the **south `MAILBOX` tab** mounts the operator mailbox (inbox + compose) — the surface the operator says to keep, view-first;
- the **agent-detail trio tab** (`Status | Mailbox | Configuration`, below the profile details) mounts the per-agent mailbox MIRROR — the duplicate to retire.

A duplicated label makes the cockpit teach two contradictory locations for "mail", and the rail duplicate spends the detail rail's scarce vertical space on a surface the south pane already owns.

## The Architectural Reality

- Rail duplicate: `apps/agentos/view/fleet/detail/Container.mjs` — trio tab at ~line 280 (`module: MailboxPane` from `../mailbox/Container.mjs`, `header: {text: 'Mailbox'}`, `reference: 'mailbox-pane'`). Live wiring that must be dispositioned, not just deleted: `pageRequest` listener (~:313), `applySnapshot` on record change (~:360), `onMailboxPageRequest` (~:395), `loadMailboxMirror` (~:424) — the read seam calling the `fleetMailboxMirror` verb with `subjectAgentId` + monotonic `mailboxReadGeneration`.
- South surface: `apps/agentos/view/fleet/cockpit/Container.mjs` pane factory, `case 'operator-mailbox'` (~:1370) mounting `mailbox/OperatorContainer.mjs`. Compose machinery exists: `mailbox/ComposeForm.mjs`, `mailbox/RecipientChip.mjs`, `mailbox/RecipientChipList.mjs` (the "chip field" the operator recalls).
- Design home for the mailbox pane's look: #20 (FM pane information design: mailbox, memories and catch-up as designed views).

## The Fix

1. Remove the Mailbox tab from the agent-detail trio (`detail/Container.mjs`), leaving `Status | Configuration`.
2. Disposition the per-agent mirror read path deliberately: the south mailbox view is the one mailbox surface, so the roster-selection-scoped mirror (subjectAgentId read via `fleetMailboxMirror`) either re-homes there as a selection-driven filter/scope, or is consciously retired with a receipt that the south view covers the read need. The implementer decides WITH #20's design direction — this ticket only forbids leaving the wiring dangling or silently dead.
3. Compose split-out is explicitly NOT this ticket: "write mail" as its own south tab stays an open design question under #20 (operator: "open minded"; current compose "still super ugly").
4. No dangling `mailbox-pane` reference, listener, or dead import remains.

## Acceptance Criteria

- [ ] Exactly one surface labeled "Mailbox" exists in the cockpit (the south pane).
- [ ] The agent-detail trio renders `Status | Configuration` only; no dangling reference/listener/import from the removed tab.
- [ ] The per-agent mirror read need is dispositioned with a receipt: either reachable from roster selection via the south view (witnessed: select agent → scoped mail), or its retirement is stated in the PR body with the covering-surface evidence.
- [ ] Unit run via the repo's custom Playwright configs (`npm run test-unit`); no default `npx playwright test`.
- [ ] Both themes render the surviving surfaces unchanged.

## Out of Scope

- Compose/"write mail" redesign and its possible own south tab — #20.
- Mailbox pane visual design — #20.
- Activity-stream visual contracts — #3.
- Agent-detail rail information architecture — #23.

## Related

- Parent: #10 (Epic: FM cockpit UI/UX — Lane B). Siblings: #20 (design home), #23 (rail IA), #3 (visual contracts).

Live latest-open sweep: checked latest 30 open institution issues + the A2A herd window at 2026-08-28T10:0xZ; no equivalent found (#20 owns pane design, no dedup/removal ticket exists).

Origin Session ID: 55add047-b483-449f-b194-dce9a0df30d4

Retrieval Hint: "mailbox tab duplication detail trio south pane view-first fleetMailboxMirror disposition"


## Timeline

- 2026-08-28T10:06:27Z @neo-fable-clio added the `bug` label
- 2026-08-28T10:06:27Z @neo-fable-clio added the `agent-os` label
- 2026-08-28T10:06:27Z @neo-fable-clio added the `ai` label
- 2026-08-28T10:06:27Z @neo-fable-clio added the `design` label
- 2026-08-28T11:11:09Z @tobiu cross-referenced by PR #31
- 2026-08-28T11:19:23Z @neo-fable-clio cross-referenced by PR #32
- 2026-08-28T11:22:28Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T11:42:04Z @neo-fable-clio cross-referenced by #20
- 2026-08-28T11:42:40Z @neo-fable-clio cross-referenced by PR #33
- 2026-08-28T12:52:22Z @tobiu referenced in commit `61dc67e` - "fix(agentos): retire the agent-detail Mailbox tab — the south pane owns the view (#30)"
- 2026-08-28T13:25:00Z @tobiu referenced in commit `42bf989` - "fix(agentos): finish the semantic retirement of the detail mailbox host (#30)"
- 2026-08-28T13:43:08Z @neo-fable-clio referenced in commit `22ddb64` - "fix(agentos): retire the agent-detail Mailbox tab — the south pane owns the view (#30)"
- 2026-08-28T13:43:08Z @neo-fable-clio referenced in commit `0c0affd` - "fix(agentos): finish the semantic retirement of the detail mailbox host (#30)"
- 2026-08-28T13:51:24Z @tobiu referenced in commit `5acdbd3` - "docs(agentos): the mirror re-entry gate is S5 grants, not viewer ingress (#30)"
- 2026-08-28T13:53:26Z @tobiu referenced in commit `9534ffc` - "chore(agentos): restamp baseline inputs for the S5 wording pass (#30)"
- 2026-08-28T14:02:21Z @tobiu referenced in commit `512c5c4` - "Merge pull request #33 from neomjs/agent/30-retire-detail-mailbox-tab

fix(agentos): retire the agent-detail Mailbox tab — the south pane owns the view (#30)"
- 2026-08-28T14:02:21Z @tobiu closed this issue
- 2026-08-28T16:17:59Z @neo-fable-clio cross-referenced by PR #36

