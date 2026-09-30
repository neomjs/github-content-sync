---
id: 337
title: Pane heads close on the SSOT hairline
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T08:22:05Z'
updatedAt: '2026-09-30T09:18:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/337'
author: neo-opus-grace
commentsCount: 0
parentIssue: 13
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
# Pane heads close on the SSOT hairline

> **Correction, 2026-09-30 (author).** This ticket first also asked the Memories pop-out to "go ghost". That finding is withdrawn. The window verbs wear a deliberate secondary skin: `Viewport.scss` states that windowing a pane "is a secondary cockpit action everywhere it appears", so it reads the same in pane chrome, in a vessel and in the cockpit bar's recall pair. A secondary verb beside a ghost one is hierarchy, not drift. I read the button's config and missed the rule owned by its class. The scope below is the head rule alone.

## Context

Design-owner review of #332 (`#247` AC-3), whose pane contract landed on dev via `#334`. The contract is a clear step toward the SSOT: one head grammar across the ten strip and rail panes, one inset, and verbs at the chrome scale. One residual remains against the SSOT frames `#247` AC-3 names.

## The Problem

**The head has no rule, so the Tasks pane's title and its sections read as peers.** The SSOT's `.pane-head` closes with `border-bottom: 1px solid var(--line-soft)` (`apps/agentos/design/institution-mailbox-pane.html:45`, `institution-memories-pane.html:44`), and the rendered frames draw it under `MAILBOX operator inbox` and `BIRD VIEW`. `.fm-pane-head` declares only gaps (`resources/scss/src/apps/agentos/Viewport.scss:225`). In Tasks, `.fm-tasks-section-label` (`resources/scss/src/apps/agentos/fleet/tasks/List.scss:44`) carries the title's exact treatment (chrome role, .08em, uppercase, dim ink), so `WHAT IS RUNNING` sits above `RUNNING` and `QUEUED · NEXT` with nothing to rank them (`pane-tasks.png`). Before `#334`, a bold sans title and grey bands ranked them. The contract rightly retired both, and nothing took their place.

## The Architectural Reality

- The contract is one block in `Viewport.scss` (THE PANE CONTRACT): `:where(.fm-pane)` inset, `.fm-pane-head` / `-title` / `-meta` / `-actions`. The pane heads are those of activity, catchup, goldenpath, AddAgentForm, mailbox, memories, perspectives, tasks and wake.
- The detail pane borrows the head's grammar for its four section cards (`fm-detail-pane-head`: Thought stream, Current lane, Repository, Pull requests). The SSOT rules sections from above (`.sect`, `institution-header-detail-ia.html:79`) and the cockpit frames them as cards, so they take no head rule. The detail's own head (`.fm-detail-header`) already carries the line-soft rule.
- The inset lives on the pane (12px), not the head. The rule sits on the inset, the cockpit's idiom: the detail header's rule and the activity rows' separators are inset the same way. The rail well zeroes the pane inset for the panes it frames, so there the rule spans the well's content width.

## The Fix

1. `.fm-pane-head` closes with the SSOT hairline (`1px solid var(--fm-line-soft)`) at zero weight inside the contract block, so every pane head inherits it.
2. The detail's section heads re-value it to none in their own sheet.
3. The Tasks section labels keep their treatment if the rule alone ranks them in the render; otherwise they take the SSOT band's tracking (`.band`, .14em).

## Acceptance Criteria

- [ ] AC-1 Every pane head draws the SSOT hairline, docked, revealed in the well and in a vessel window; the detail's section cards do not; goldens show both.
- [ ] AC-2 In Tasks, the pane title and its section labels read at different ranks; the Tasks goldens (720 and 240, dark and light) show it.
- [ ] AC-3 Before/after per affected pane in the PR, beside the SSOT frames.

## Out of Scope

- The window verbs' skin (see the correction above) and where window verbs live: the detail's pop-out is icon-only on the tab header's action seam, the Memories pop-out a worded verb in the pane head.
- The contract's inset and verb scale (landed with `#247`); the panes' content (`#237`, `#308`, `#309`).

## Related

Parent: #13 · Related: #247, #332, #24

Live latest-open sweep: latest 20 open Institution issues at 2026-09-30T08:21Z, no equivalent. A2A in-flight sweep (all read states, latest 15): no claim on pane heads. Memory Core sweep (pane head rule, section label treatment): no recorded decision to drop the rule. Own-assignment sweep: Institution `#11` only, unrelated.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636
Retrieval Hint: `query_raw_memories("pane head hairline rule Tasks section label second title")`


## Timeline

- 2026-09-30T08:22:05Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T08:22:06Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T08:22:07Z @neo-opus-grace added the `ai` label
- 2026-09-30T08:22:07Z @neo-opus-grace added the `design` label
- 2026-09-30T08:22:10Z @neo-opus-grace added parent issue #13
- 2026-09-30T08:23:08Z @neo-opus-grace cross-referenced by PR #332
- 2026-09-30T08:29:02Z @neo-fable-clio cross-referenced by #338
- 2026-09-30T09:02:18Z @neo-fable-clio cross-referenced by #340
- 2026-09-30T09:18:57Z @neo-opus-grace changed title from **Pane heads keep the SSOT hairline, and the Memories pop-out goes ghost** to **Pane heads close on the SSOT hairline**
- 2026-09-30T09:19:25Z @neo-opus-grace cross-referenced by PR #343
- 2026-09-30T09:26:05Z @neo-opus-grace cross-referenced by #247
- 2026-09-30T12:32:57Z @neo-fable-clio cross-referenced by #349
- 2026-09-30T13:19:29Z @neo-fable-clio cross-referenced by #351
- 2026-09-30T13:42:28Z @tobiu referenced in commit `f24caee` - "Merge pull request #343 from neomjs/grace/337-pane-head-rule

fix(agentos): pane heads close on the SSOT hairline (#337)"

