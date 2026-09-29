---
id: 309
title: Define Catch Up around meaningful changes and decisions
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-28T10:09:52Z'
updatedAt: '2026-09-29T11:46:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/309'
author: neo-gpt-emmy
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
closedAt: '2026-09-29T11:46:12Z'
---
# Define Catch Up around meaningful changes and decisions

## Context

The operator's 2026-09-28 Catch Up screenshot shows scattered period choices, oversized actions, large empty areas and technical copy (“No runtime anchor yet”, “query-time · not authority”). The subsequent clarification is the governing order: product purpose and content first, visual design afterward.

Our own team is the first acceptance environment. FM also serves peers through NL or possible APIs (MX); outbound first-time operators and teams remain a distinct journey.

## The Problem

A better arrangement of existing controls does not establish what Catch Up should tell a returning operator or peer. Its purpose overlaps Home and the Observatory unless each view's responsibility is decided once.

This is a **design/specification deliverable**, not authorization to reimplement history, synthesis or read acknowledgements.

## The Architectural Reality

- Existing design: `apps/agentos/design/institution-catchup-pane.html`.
- `view/fleet/catchup/Container.mjs` composes scope/window choices, source-owned histories, citations and explicit mark-caught-up intent.
- #244 owns Home; D#19317 owns the Observatory's product definition; #247 covers common pane headers/insets/buttons and explicitly excludes content.
- Closed #20 supplied the earlier design contract. The current operator correction reopens purpose and primary copy, not every source mechanism.

## The Fix

Revise the existing Catch Up design artifact in one change. Define which meaningful changes and decisions it presents, how an operator or peer obtains useful context, and where a follow-up action leads. Reconcile its role with Home and the Observatory before choosing the default content hierarchy.

Then design the first-use, returning-use and partially available journeys from that content. Identify data that already exists versus missing source capabilities; do not hide the latter with placeholder prose.

## Contract Ledger

| Surface | Authority | Deliverable | Evidence |
|---|---|---|---|
| Catch Up product purpose | Operator correction; existing Home/Observatory scopes | Human and peer questions, content hierarchy and walkthrough | AC-1–3 |
| History and acknowledgement | Existing source-owned histories and explicit mark intent | Design preserves scope, freshness, evidence links and deliberate acknowledgement | AC-2–3 |

## Acceptance Criteria

- [ ] AC-1 State the human and peer outcomes and the division of responsibility among Home, Catch Up and Observatory; each primary content element answers a named question.
- [ ] AC-2 Map the proposed content to real source capabilities and preserve distinct loading, empty, unavailable and partial states. Record missing data and unresolved choices.
- [ ] AC-3 Render and walk first-use and returning-use designs in both themes. A person can identify what changed, why it matters and the next available action; a peer's compact answer has equivalent source/freshness meaning.
- [ ] AC-4 Keep automatic acknowledgement, arbitrary period selection and new synthesis/ranking outside the design unless separately discussed and approved. Derive implementation work only after the product shape is settled.

## Out of Scope

Brain API changes, new historical synthesis, automatic mark-caught-up, shared chrome implementation (#247), and graph implementation.

## Avoided Traps

Reproducing the current controls with nicer CSS; building three views for the same returning-user job; silently equating an unavailable source with no changes.

Decision Record impact: none proposed; source/API changes remain open.
Structure map: N/A — existing HTML design artifact, no new module.
Related: #13 (parent), #20, #244, #247, neomjs/neo#19317.

unowned-rationale: current capture for peer self-selection after the graph-first milestone, without claiming another implementation.

Freshness sweep: latest 20 open issues, #20/#244/#247 bodies and recent all-state A2A checked 2026-09-28; no equivalent content-definition leaf. Own Institution assignments: none open after #302 merged. MC `4359bb4b-2de2-441d-8644-e7013a35ded1` records the purpose-before-presentation correction.

Origin Session ID: 23b22a41-52ac-4e6c-8d80-23d54054c48c
Retrieval Hint: `Catch Up meaningful changes decisions Home Observatory product purpose`
Authored by Emmy (GPT-6 Astra, Codex).

## Timeline

- 2026-09-28T10:09:54Z @neo-gpt-emmy added the `enhancement` label
- 2026-09-28T10:09:55Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-28T10:09:55Z @neo-gpt-emmy added the `ai` label
- 2026-09-28T10:09:55Z @neo-gpt-emmy added the `design` label
- 2026-09-28T10:09:58Z @neo-gpt-emmy added parent issue #13
- 2026-09-28T10:11:17Z @neo-gpt-emmy cross-referenced by #244
- 2026-09-28T10:11:21Z @neo-gpt-emmy cross-referenced by #247
- 2026-09-28T17:32:35Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-28T17:45:50Z @neo-opus-vega cross-referenced by PR #318
- 2026-09-28T18:16:40Z @neo-opus-vega referenced in commit `386f8c4` - "chore(deps): drop the dompurify override monaco-editor 0.57 made redundant (#309)"
- 2026-09-29T10:08:53Z @neo-opus-vega referenced in commit `3fbaccb` - "docs(agentos): Catch Up withholds the close across a failed read and links every folded record (#309)

Frame 2 no longer offers Done reading while pull requests could not be
read: today's markFleetCaughtUp records the rendered end regardless, so
a close would start the next window past changes the pane never showed.
The target contract is gap 6, and the peer's answer says whether the
close is offered.

The dependency and six-more folds link each pull request, and the live
feed and cited memories are marked static and name their in-app
targets (the cockpit's stream and memories panes). Section 2 now says
Catch Up leads to the reason instead of stating it; reading the resolved
ticket's own title as a record joins gap 5."
- 2026-09-29T10:09:15Z @neo-opus-vega referenced in commit `e8f4b9a` - "docs(agentos): renders of the Catch Up design at 3fbaccb (#309)"
- 2026-09-29T11:29:48Z @neo-preview cross-referenced by PR #317
- 2026-09-29T11:46:12Z @tobiu referenced in commit `d168faa` - "Merge pull request #318 from neomjs/vega/309-catchup-definition

docs(agentos): Catch Up is defined by what changed and where it leads (#309)"
- 2026-09-29T11:46:13Z @tobiu closed this issue

