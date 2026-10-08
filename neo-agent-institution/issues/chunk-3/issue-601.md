---
id: 601
title: Accounts body gestures start a popup drag
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-08T00:47:06Z'
updatedAt: '2026-10-08T00:47:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/601'
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
blocking: []
---
# Accounts body gestures start a popup drag

## Context

The operator reported on 2026-10-08 that a small drag anywhere in the Accounts pane opens a popup. This interrupts normal work in the agent list and detail form. The operator explicitly separated this from the popup-close restoration defect.

Observed installed candidate: Institution `fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e`, Engine `82bc6158444306e0c342e8cda480e77158c9fedb`. Read-only inspection confirms the Accounts wrapper carries `neo-draggable` and its dashboard sort zone accepts `.neo-draggable`.

## The Problem

Body interactions can be admitted as pane movement because the same marker identifies the sortable item and its supposed handle. Clicking, selecting text and using controls must not accidentally detach the view.

Design authority: operator, 2026-10-08: “dragging a view BODY should not be possible => header based dragging wins.”

## The Architectural Reality

Accounts is a legacy `Neo.dashboard.Panel` inside its own `Neo.dashboard.Container`, separate from the cockpit DockLayout. See [Accounts header](https://github.com/neomjs/neo-agent-institution/blob/fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e/apps/agentos/view/accounts/Panel.mjs) and [dashboard host](https://github.com/neomjs/neo-agent-institution/blob/fd958fba1e7d492a5ae2b9b283df1f65e5cc1a6e/apps/agentos/view/Viewport.mjs). The header explicitly has `neo-draggable`; Engine `src/draggable/container/DragZone.mjs#adjustItemCls` also adds that class to sortable wrappers. Dashboard SortZone defaults its handle selector to the same class.

## The Fix

Configure the Accounts dashboard's existing sort zone with a selector that matches only its intended header, and give that header a distinct handle marker if needed. Preserve header-based sorting and tear-out. Keep this bounded to the consumer; changing generic dashboard defaults needs its own compatibility evidence.

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Accounts drag admission | Operator decision above; existing dashboard handle selector | Only its header starts pane movement | Body remains ordinary interactive content | Header/host intent in existing JSDoc | Native gesture test with header positive control and body negative controls |

Decision Record impact: none; restores the requested interaction boundary using existing Engine configuration. Structure-map command is not hosted in Engine; source placement stays in existing Accounts/Viewport files, with no new service or module.

## Acceptance Criteria

- [ ] A drag beginning in body whitespace, list text, detail text or form controls does not start the dashboard drag or open a popup.
- [ ] A deliberate Accounts header drag still starts pane movement and can tear out.
- [ ] Body selection, scrolling and controls remain usable.
- [ ] Regression coverage fails on the broad body selector and passes with the header boundary; an installed receipt records the actual candidate separately from source tests.

## Out of Scope

Popup-close restoration/duplication; generic DockLayout changes; removing Accounts popout capability.

## Related and discovery

Related: #12 and #505. The separately tracked Engine popup-close defect must remain independently testable with a valid header drag.

Live latest-open sweep: latest 20 created-descending Institution and Engine issues at 2026-10-08T00:46Z, no equivalent. All-state latest 30 A2A messages: no competing Accounts claim. Historical org search `Accounts popup`: no match. MC sweep `Accounts pane body dragging popup close duplicated view`: three results, earlier popup/dock incidents but no matching header-boundary decision. Own-assignment sweep: Institution none; Engine only `#19446`, a different surface.

unowned-rationale: captured for the post-grid FM lane; Sophie retains acceptance coordination, but has no Institution checkout in this session and is completing Engine `#19446` first.

Origin Session ID: e6ce4d70-a7ff-454e-996d-e7c25efdf4cf

Retrieval Hint: "Accounts body drag header-only handle popup"

## Timeline

- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `bug` label
- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-08T00:47:07Z @neo-gpt-sophie added the `ai` label
- 2026-10-08T00:47:08Z @neo-gpt-sophie added the `design` label
- 2026-10-08T00:47:54Z @neo-gpt-sophie cross-referenced by #19462
- 2026-10-08T00:50:38Z @neo-gpt-sophie cross-referenced by #12

