---
id: 247
title: 'The reading strip''s panes share one head, one inset, one button scale'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-09-26T09:34:32Z'
updatedAt: '2026-09-26T09:34:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/247'
author: neo-opus-ada
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
# The reading strip's panes share one head, one inset, one button scale

## Context

The operator, 2026-09-26, after naming the perspective bar, Home, Accounts and the legend: *"this is by far not all."* A headless walk of dev (1280×800, dark) through the reading strip and the right rail shows the next class: every pane draws its own head and its own buttons.

- **Action buttons at the wrong scale.** "Refresh" (Golden Path, Tasks, Memories, Catch up) and "Read routes" (Wake routes) render at about twice the strip's type size, in bold, on no surface, so each reads like a heading. "Pop out memories" is a slab.
- **Four head styles.** A sans heading ("Golden Path"), a heading with a mono sub-line ("What is running", "What they remember", "Since you last looked"), a small plain label ("A2A Mailbox"), and a mono caps line ("GOLDEN PATH · GRAPH").
- **Two insets.** Golden Path, Tasks, Mailbox and Catch up start at the strip's edge; Memories is inset by 16 px.
- **An oversized mailbox recipient field** ("All agents (broadcast)"), taller than the strip's other controls.

## The Problem

The panes use `Neo.button.Base` with `ui: 'ghost'`. A grep of `resources/scss/src/apps/agentos` finds ghost rules only in the roster and card sheets, which scope their own "ghost verb" contract, so the strip's buttons fall back to the engine theme's button size. The heads are hand-made per pane. #13 names this class: the component skin layer with a button hierarchy (primary / secondary / ghost) on the loaded tokens.

## The Architectural Reality

- The buttons: `apps/agentos/view/fleet/goldenpath/Container.mjs`, `memories/Container.mjs`, `tasks/Container.mjs`, `catchup/Container.mjs` ("Refresh"), `wake/Container.mjs` ("Read routes"), `cockpit/VesselContainer.mjs` ("Pop out memories"), each `{module: Button, ui: 'ghost', iconCls: ...}`.
- The engine's ghost skin: `neo.mjs/resources/scss/src/button/Base.scss` `.neo-button-ghost` (theme variables only, no size).
- The scoped precedent: `resources/scss/src/apps/agentos/fleet/roster/Container.scss` (the sort/filter "quiet ghost verbs").
- Tokens: `apps/agentos/resources/tokens.css` (`--fm-text-chrome`, `--fm-space-*`, `--fm-ink-dim`).

## The Fix

1. One cockpit ghost-button rule at chrome scale (`--fm-text-chrome`, icon plus word, no slab), applied cockpit-wide rather than per pane; the roster's scoped copy folds into it.
2. One pane head: a title plus an optional mono meta line plus a right-aligned action slot, as a shared component or SCSS partial every strip and rail pane uses.
3. One inset for strip and rail panes, from the `--fm-space-*` scale.
4. The mailbox recipient field uses the cockpit's field scale.

## Acceptance Criteria

- [ ] AC-1 Every strip and rail pane's action button renders at the chrome scale; there is one ghost rule in the cockpit SCSS (grep in the PR).
- [ ] AC-2 Golden Path, Tasks, Memories, Mailbox, Catch up, Route graph and Wake routes render the same head structure and inset (component arm or visual goldens).
- [ ] AC-3 Before/after screenshots per pane in the PR beside the SSOT frame; the design owner's review.

## Out of Scope

- The panes' content and data (#237 governs sample data).
- Moving panes between the strip and the rail (the Observatory's own ticket).

## Related

#13 (parent: design conformance) · #24 (view layer conforms to the component library) · #10

unowned-rationale: design conformance, claimable (#13's steward note: the cockpit owner, with Grace's design review); its author holds #241 and #242.

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): no claim on strip chrome. Memory Core sweep: none. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("reading strip pane head ghost button chrome scale cockpit conformance")`

## Timeline

- 2026-09-26T09:34:33Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T09:34:33Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:34Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:34Z @neo-opus-ada added the `design` label
- 2026-09-26T09:35:08Z @neo-opus-ada added parent issue #13
- 2026-09-26T10:04:50Z @neo-fable-clio cross-referenced by #249

