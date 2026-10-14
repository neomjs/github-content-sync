---
id: 687
title: Let users set target positions for external harness windows
state: OPEN
labels:
  - enhancement
  - ai
  - design
  - model-experience
assignees: []
createdAt: '2026-10-14T07:44:11Z'
updatedAt: '2026-10-14T07:44:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/687'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 9
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
# Let users set target positions for external harness windows

## Context
Tobi requested this product improvement on 2026-10-14: Fleet Manager should let users, including peers operating the UI, define harness target positions. His current setup has eight harnesses in two rows on an external monitor; Emmy occupies the bottom-right slot.

**Priority: lower-priority backlog.** Preserving an already chosen window position matters now; this configurable UI must not delay that separate reliability investigation.

## The Problem
The operator arranges external harness windows manually. Fleet has no explicit product control for expressing the desired per-seat desktop layout. A launched process and a correctly placed window are different outcomes.

The reported Emmy-only restoration failure is a motivating observation, not a proven Fleet geometry bug. Do not use this feature as its root-cause explanation.

## The Architectural Reality
Institution owns Fleet's UI. Brain's `ai/services/fleet/deriveHarnessLaunchSpec.mjs` launches Codex Desktop with a seat profile and project; `FleetLifecycleService.mjs` owns the process lifecycle. The inspected paths do not supply target coordinates. Institution's `VesselContainer.mjs` positions Fleet's own Neo tear-out windows; it is not an external harness placement service.

Keep requested placement scoped to the local host/display arrangement and the exact managed seat. The cockpit expresses intent; a supported host operation must observe the resulting native window before reporting success. Existing app-specific session/profile restoration remains distinct.

## The Fix
Add a small Fleet control for selecting a managed seat's target display and position/size, with a way to adopt the seat's current window rectangle. Humans and peers use the same UI and underlying action. Retain the choice across Fleet relaunch and apply it at that seat's next start or an explicit Apply action.

At intake, choose the smallest supported host adapter and storage owner from current source. Do not implement positioning by editing undocumented vendor profile files. Keep the first delivery bounded to a supported desktop platform; expose unsupported combinations honestly.

## Contract Ledger
| Proposed surface | Authority | Behavior | Fallback | Docs and evidence |
|---|---|---|---|---|
| Per-seat target-position control | Explicit user/peer choice through Fleet UI | Save, edit, clear, or adopt the current native rectangle | No target preserves normal harness restoration | UI help; keyboard and Neural Link journey |
| Apply/start placement result | Exact managed process/window and current display work area | Distinguish requested, observed applied, and refused/unavailable | Missing display, permission, window, or platform yields a reason; no silent success | Native bounds readback; wrong-seat and missing-display controls |

## Decision Record impact
None prescribed at filing. Preserve the cockpit's client/host-authority split and the shared human/peer UI boundary; intake must identify any required authority amendment before implementation.

## Acceptance Criteria
- [ ] A user can set, read, change, clear, and adopt the current rectangle for one managed seat through Fleet.
- [ ] A peer can use the same control/action through existing UI tooling; there is no separate agent-only layout system.
- [ ] Saved targets survive Fleet relaunch. Start and explicit Apply address only the intended seat/window; other seats remain unchanged.
- [ ] Observed native bounds confirm applied placement. A missing display or unsupported/denied operation retains the requested target and shows a useful reason instead of claiming success.
- [ ] No-target behavior and manual moves remain usable; this is not a continuous force-tiling loop.
- [ ] Validate a two-row layout with independent seats and a changed/disconnected-display control on the declared supported host.

## Out of Scope
A general OS window manager; arranging unrelated apps; forcing all platforms into one adapter; vendor profile surgery; the separate current-position restoration diagnosis; delaying avatar work or Engine 13.3 for this enhancement.

## Avoided Traps
Fixed global screen coordinates without display validation; matching windows by title alone; treating process spawn as placement success; silently overwriting manual arrangement; reusing Neo pane placement as authority over another app.

## Related
Parent: #9 (Fleet keeper-view and human/peer controls). Context: #12 (native-shell UX); neomjs/neo-agent-brain#571 (managed seat launch).

unowned-rationale: Parked at Tobi's explicit lower priority; no implementation lane is reserved.

Sweep: current Institution/Brain searches for harness/window positions found cockpit and Neo tear-out work, not an equivalent external-harness control. Institution's sole Emmy-assigned issue, `#42`, is unrelated view-layer debt. KB returned internal cockpit/tear-out matches; MC returned older display-operation context, no matching product decision. Brain scoped structure map identifies the existing Fleet launch/lifecycle owner; no new module placement is prescribed. Latest 20 open Institution issues and all-read-state A2A claims were checked at 2026-10-14 07:41 UTC immediately before creation; no equivalent found.

Origin Session ID: 05b71d01-be36-41f5-8d86-6c898089c6ec
Retrieval Hint: external harness window positions; eight harnesses two rows; keep chosen position versus explicit target layout.

Authored by Emmy (GPT-6, Codex Desktop).

## Timeline

- 2026-10-14T07:44:13Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-14T07:44:13Z @neo-gpt-emmy added the `ai` label
- 2026-10-14T07:44:13Z @neo-gpt-emmy added the `design` label
- 2026-10-14T07:44:13Z @neo-gpt-emmy added the `model-experience` label
- 2026-10-14T07:44:45Z @neo-gpt-emmy added parent issue #9
- 2026-10-14T08:07:50Z @neo-gpt-emmy cross-referenced by #12

