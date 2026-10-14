---
id: 688
title: Add repositories with Enter in Accounts
state: CLOSED
labels:
  - enhancement
  - accessibility
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-14T09:59:50Z'
updatedAt: '2026-10-14T10:30:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/688'
author: neo-gpt-emmy
commentsCount: 1
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
closedAt: '2026-10-14T10:30:08Z'
---
# Add repositories with Enter in Accounts

## Context
Tobi reported on 2026-10-14 that entering a repository in Accounts requires clicking Add; pressing Enter in the repository field does nothing.

## The Problem
The installed Repositories card has a single-line `field-repo` input but no keyboard submit handler. Source at `apps/agentos/view/fleet/detail/AgentReposContainer.mjs` wires only the Add button to `onAddClick`.

## The Architectural Reality
Design authority: Tobi's requested Accounts interaction, “hitting the enter key inside a repo name form field does not work yet => we need to click the add button.”
The card emits `configIntent`; Accounts owns the existing `setRepos` round-trip and canonical readback. `onAddClick` already normalizes the slug and refuses empty/pending operations. This is the existing-account Repositories card, not AddAgentForm.

## The Fix
Bind Enter on this field to the existing add path using Neo's DOM-event handling. Retain the existing validation, pending guards and accepted/refused presentation.

| Surface | Authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| Accounts repository input | Operator request; `onAddClick` | Enter submits the same whole-list intent as Add | Empty or pending input produces no write; server refusal retains input | Real Fleet wire keyboard journey and registry readback |

## Acceptance Criteria
- [ ] Enter from the repository input adds a valid repository through the existing round-trip and clears the field after acceptance.
- [ ] Empty/pending submissions remain guarded; duplicate rejection retains the slug and canonical list.
- [ ] Add-click and Remove continue working; Enter handling stays local to the input.

## Out of Scope
Other forms, repository defaults, provisioning policy, live account-list maintenance and harness launches.

## Avoided Traps
No app-wide keyboard listener, synthetic click, duplicated mutation path or optimistic Store-only success.

Decision Record impact: none; existing UI event and registry ownership retained.
Related: #642.

Sweeps: latest 20 open Institution issues and recent all-state A2A checked immediately before filing; no equivalent. Exact all-state repository/Enter issue search had no keyboard owner. MC queries for Accounts repository Enter and AgentReposContainer found no prior decision. Own-assignment sweep found only #42's separate investigation. No new module or service boundary.

Origin Session ID: 05b71d01-be36-41f5-8d86-6c898089c6ec
Authored by Emmy (GPT-6, Codex Desktop).

## Timeline

- 2026-10-14T09:59:50Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-14T09:59:51Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-14T09:59:52Z @neo-gpt-emmy added the `accessibility` label
- 2026-10-14T09:59:52Z @neo-gpt-emmy added the `ai` label
### @neo-gpt-emmy - 2026-10-14T10:00:59Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-gpt-emmy`
**New assignees:** `neo-gpt-emmy`
**Reason:** Confirm the sole existing atomic self-assignment from issue creation; no owner change.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-14T10:06:36Z @neo-gpt-emmy cross-referenced by PR #689
- 2026-10-14T10:09:21Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-14T10:30:08Z @tobiu referenced in commit `56eff91` - "feat(accounts): add repositories with Enter (#688) (#689)"
- 2026-10-14T10:30:08Z @tobiu closed this issue

