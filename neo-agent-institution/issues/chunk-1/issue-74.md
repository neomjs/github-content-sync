---
id: 74
title: 'The define-agent zone reveals empty: the S5 form never materializes from the rail or the CTA'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-fable-clio
createdAt: '2026-09-01T23:24:20Z'
updatedAt: '2026-09-02T14:40:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/74'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-02T14:40:46Z'
---
# The define-agent zone reveals empty: the S5 form never materializes from the rail or the CTA

Sub-issue of #10. Found by the widened Neural Link battery (#73) — `AddAgentJourneyNL` is red at its first product step.

## Context

The S5 define-agent zone reveals EMPTY on current `dev` (`0b7542b`, 2026-09-01 23:30Z, live cockpit at 1280×720): clicking the rail tab "Add agent" opens the reveal overlay with its title, pin and an empty `neo-dashboard-dock-reveal-pane-slot` — no `.fm-add-agent-form`, no `fm-add-*` element anywhere in the document after 3s. The witness sees the same from the other entry: on an authoritative-empty roster the bootstrap CTA "Add your first agent" fires `addAgentRequest` → `onAddAgentRequest` un-hides `defineAgent` → nothing materializes, `.fm-add-agent-form` never appears within 30s. Both entry points end in the same empty zone.

## The Problem

This is the product's first-run path — the only way an empty institution gets its first agent — and it is silently broken: the zone opens, the form does not. Nothing in CI can see it (Neural Link witnesses do not run there), and the two witnesses that would have caught it sat outside the battery glob until #73.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/Container.mjs` (`case 'define-agent'`, ~line 752): the pane config was `module: () => import('../instances/AddAgentForm.mjs')` — deliberately LAZY ("the define-agent zone opens on explicit intent only"), since `cbd7fc2` (#50).
- The cause, read at the seam (comment below, 2026-09-01 23:35Z): the lazy config survives the projection (`LayoutAdapter.decorateItemConfig` spreads it, the function intact) and the reveal mounts it with `paneSlot.add(config)` → `container.Base#insert` → `createItem`, which recognises a lazy item and keeps it as a plain Object with `vdom.removeDom: true`. A lazy `module` function is loaded by the **card layout on tab activation**; the reveal overlay's pane slot is a plain vbox container, and a rail-only auto-hidden item never takes the tab-body path — so the item sat in the slot as an unrendered Object.
- `CockpitDockDocument.mjs:64`: `defineAgent: {componentRef: 'define-agent', title: 'Add agent', kind: 'tool', autoHidden: true}` — the item is on the secondary rail, auto-hidden.

## The Fix

- Import `AddAgentForm` statically in the cockpit's resolver: the zone opens on explicit intent either way, and a lazy chunk buys nothing the dev build can measure; the Engine's synchronous resolver contract stays untouched. (Teaching the reveal slot to load lazy items would be an Engine leaf if anyone wants the lazy split back.)
- `AddAgentJourneyNL` is the witness for the zone: the CTA → S5 form → readback → settle steps; the rail-tab entry gets a live read.

## Acceptance Criteria

- [ ] Rail tab "Add agent" reveals the pane with `.fm-add-agent-form` mounted (DOM read on the live cockpit).
- [ ] The bootstrap CTA on an empty roster reveals the same form; `AddAgentJourneyNL` runs green through the form, the readback-confirmed status, the PAT settle and the roster's `Fleet · 1 agents` graduation.
- [ ] The cause is named in the PR (projection decorate vs runtime add vs import path), with the red-first receipt.

Amended 2026-09-02 00:50Z after the fix landed: the journey's LAST step — the graduated agent's card rendering into a roster that was empty — is a different defect one level below the cockpit (pooled cards re-seated by reference after their DOM left with the rows) and is split out as #76 with the full analysis; this ticket resolves the zone.

## Out of Scope

- The battery membership and the other stated reds — #73.
- The graduated card after an empty roster — #76.
- Any redesign of the S5 zone's content.

## Related

Parent: #10. Found by #73 / PR #70's disclosure record. The lazy import: `cbd7fc2` (#50). Successor: #76.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-01T23:23Z plus a title search for "add agent / define-agent / S5" — no equivalent (nearest: #12, the native shell's first-run specification, a different surface); A2A lane-claims over the last 30 messages cover Engine dock leaves, skills and Brain — none on this zone; Memory Core raw query on the define-agent zone returned nothing.

Origin Session ID: f353cda0-1b36-49e4-9b73-3f7aec1896e0

Retrieval Hint: `define-agent zone reveal empty pane slot lazy module import AddAgentForm dock projection`

📜 Clio


## Timeline

- 2026-09-01T23:24:20Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-01T23:24:22Z @neo-fable-clio added the `bug` label
- 2026-09-01T23:24:22Z @neo-fable-clio added the `agent-os` label
- 2026-09-01T23:24:22Z @neo-fable-clio added the `ai` label
- 2026-09-01T23:24:23Z @neo-fable-clio added the `regression` label
- 2026-09-01T23:30:31Z @neo-fable-clio cross-referenced by PR #75
### @neo-fable-clio - 2026-09-01T23:32:39Z

**Cause, read at the seam (2026-09-01 23:35Z, before any fix):** the lazy config survives the projection — `LayoutAdapter.decorateItemConfig` spreads it (`{...component}`, the `module` function intact) and the reveal mounts it with `paneSlot.add(config)` → `container.Base#insert` → `createItem`, which recognises a lazy item (`lazyLoadItem = module && !module.isClass && Neo.isFunction(module)`) and keeps it as a plain Object with `vdom.removeDom: true` (`container/Base.mjs` ~442-462). Nothing then loads it: a lazy `module` function is resolved by the **card layout on activation** (the tab-body path), and the reveal overlay's pane slot is a plain vbox container — so the define-agent item sits in the slot as an unrendered Object forever. The tab-body path never applies because the item is `autoHidden` (rail-only). That is why the slot is empty from the rail and from the CTA alike.

Fix shape, narrowed: import `AddAgentForm` statically in the cockpit's resolver (the zone opens on explicit intent anyway; the lazy chunk has no measurable benefit in this build), or teach the reveal slot to load lazy items — the first is a two-line cockpit change and keeps the Engine's synchronous resolver contract; the second is an Engine leaf if anyone wants the lazy split back.

📜 Clio

- 2026-09-02T00:48:45Z @neo-fable-clio cross-referenced by #76
- 2026-09-02T00:49:36Z @neo-fable-clio cross-referenced by PR #77
- 2026-09-02T01:02:45Z @neo-fable-clio referenced in commit `d36baa5` - "chore(test): merge dev and refresh the visual baseline stamp after the define-agent fix (#74)"
- 2026-09-02T01:23:48Z @neo-fable-clio cross-referenced by PR #18061
- 2026-09-02T08:54:56Z @neo-fable-clio cross-referenced by #18062
- 2026-09-02T09:05:53Z @neo-fable-clio cross-referenced by PR #18063
- 2026-09-02T09:23:22Z @neo-fable-clio referenced in commit `88f0bd7` - "fix(agentos): the define-agent zone stays lazy — the engine's rail loads it on reveal (#74)"
- 2026-09-02T09:29:15Z @neo-fable-clio cross-referenced by #78
- 2026-09-02T09:30:51Z @neo-fable-clio cross-referenced by #73
- 2026-09-02T11:40:26Z @neo-fable-clio cross-referenced by PR #79
- 2026-09-02T12:24:10Z @neo-fable-clio cross-referenced by #80
- 2026-09-02T12:31:10Z @neo-fable-clio referenced in commit `dc2af7a` - "fix(agentos): the define-agent zone materializes — a static module for a reveal-only pane (#74)

The cockpit resolved the S5 pane with a lazy module function. A lazy module is loaded by the card layout on tab activation, a path an auto-hidden rail item never takes: the reveal overlay's pane slot is a plain container that keeps a lazy item as an unrendered object, so the zone opened empty from the rail tab and from the bootstrap CTA. The module is static now; the zone still opens on explicit intent only. The journey witness's Start assertion learns the realm's protocol offer (the keyboard witness's shape); it now runs through form, readback, settle and roster graduation and stops at the graduated card — the successor ticket's seam."
- 2026-09-02T12:31:10Z @neo-fable-clio referenced in commit `9f65e07` - "chore(test): merge dev and refresh the visual baseline stamp after the define-agent fix (#74)"
- 2026-09-02T12:31:11Z @neo-fable-clio referenced in commit `17530fa` - "fix(agentos): the define-agent zone stays lazy — the engine's rail loads it on reveal (#74)"
- 2026-09-02T12:31:11Z @neo-fable-clio referenced in commit `0c6f522` - "chore(test): refresh the visual baseline stamp after the rebase onto dev (#74)"
- 2026-09-02T12:55:53Z @neo-fable-clio referenced in commit `3d345eb` - "chore(deps): bump the engine pin to dev@961ac9cd11 — the rail loads a lazy module item on reveal, the dock host opens the geometry stream (#74)"
- 2026-09-02T13:06:55Z @neo-fable-clio cross-referenced by #18084
- 2026-09-02T13:14:36Z @neo-fable-clio cross-referenced by PR #18086
- 2026-09-02T14:01:03Z @neo-fable-clio referenced in commit `1b8d966` - "chore(deps): bump the engine pin to dev@fabb61f001 — insert parks a lazy module config, the reveal chrome floor (#74)"
- 2026-09-02T14:18:08Z @neo-fable-clio referenced in commit `89e7bb8` - "fix(agentos): the define-agent zone materializes — a static module for a reveal-only pane (#74)

The cockpit resolved the S5 pane with a lazy module function. A lazy module is loaded by the card layout on tab activation, a path an auto-hidden rail item never takes: the reveal overlay's pane slot is a plain container that keeps a lazy item as an unrendered object, so the zone opened empty from the rail tab and from the bootstrap CTA. The module is static now; the zone still opens on explicit intent only. The journey witness's Start assertion learns the realm's protocol offer (the keyboard witness's shape); it now runs through form, readback, settle and roster graduation and stops at the graduated card — the successor ticket's seam."
- 2026-09-02T14:18:09Z @neo-fable-clio referenced in commit `ce673da` - "chore(test): merge dev and refresh the visual baseline stamp after the define-agent fix (#74)"
- 2026-09-02T14:18:09Z @neo-fable-clio referenced in commit `d9be490` - "fix(agentos): the define-agent zone stays lazy — the engine's rail loads it on reveal (#74)"
- 2026-09-02T14:18:09Z @neo-fable-clio referenced in commit `5a55acd` - "chore(test): refresh the visual baseline stamp after the rebase onto dev (#74)"
- 2026-09-02T14:18:09Z @neo-fable-clio referenced in commit `5b0472b` - "chore(deps): bump the engine pin to dev@961ac9cd11 — the rail loads a lazy module item on reveal, the dock host opens the geometry stream (#74)"
- 2026-09-02T14:18:09Z @neo-fable-clio referenced in commit `9419e97` - "chore(deps): bump the engine pin to dev@fabb61f001 — insert parks a lazy module config, the reveal chrome floor (#74)"
- 2026-09-02T14:40:46Z @tobiu referenced in commit `577e03b` - "Merge pull request #77 from neomjs/agent/74-define-agent-zone

fix(agentos): the define-agent zone stays lazy — the engine loads it on reveal (#74)"
- 2026-09-02T14:40:46Z @tobiu closed this issue
- 2026-09-02T14:46:01Z @neo-fable-clio cross-referenced by #81
- 2026-09-02T14:52:33Z @neo-fable-clio cross-referenced by PR #82
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-02T15:47:41Z @neo-fable-clio cross-referenced by #84
- 2026-09-02T17:12:50Z @neo-fable-clio cross-referenced by PR #86
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T11:21:52Z @neo-fable-clio cross-referenced by PR #95
- 2026-09-12T10:10:47Z @neo-fable-clio cross-referenced by #120

