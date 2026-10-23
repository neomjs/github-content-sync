---
id: 788
title: 'Every engine Splitter in the cockpit wears the engine theme, not the dock''s splitter tokens'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-23T15:11:57Z'
updatedAt: '2026-10-23T16:46:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/788'
author: neo-fable-clio
commentsCount: 0
parentIssue: 505
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
# Every engine Splitter in the cockpit wears the engine theme, not the dock's splitter tokens

## Context

Operator report (2026-10-23, on his installed Fleet Manager): the Memories view's splitter is blue and looks odd next to the main view's green splitter. Code read at `origin/dev` 024a7a5586 confirms a class, not one view: three cockpit views compose the engine's `Neo.component.Splitter` inside the cockpit root and none of them gives it the cockpit's tokens; a fourth (the Observatory) sits outside the cockpit root and already carries its own copy of the dock values (build finding, below).

Live latest-open sweep: checked the latest 20 open issues at 2026-10-23 14:5xZ; no equivalent (#505's closed leaves #506/#713/#715 touched the Memories pane's reader and rail, not its splitter). A2A claim sweep (last 14 broadcasts): none. Memory Core rationale sweep: no decision on splitter styling recovered. Own-assignment sweep: #505/#507 (mine) are the parents, not duplicates.

## The Problem

The dock's splitters are tokenised in `resources/scss/src/apps/agentos/fleet/cockpit/Container.scss` (on dev 024a7a5586 at lines 317-324, on the `.neo-dashboard-dock-splitter` rule inside `.fm-fleet-cockpit`: `--dock-splitter-background: color-mix(in srgb, var(--fm-line-soft) 55%, transparent)`, hover and active `color-mix(in srgb, var(--fm-signal) 45%, transparent)`), which is the green the operator likes. The engine `Splitter` instances inside the panes read the engine theme instead: `resources/scss/src/component/Splitter.scss` paints `--splitter-background-color` at rest and `--splitter-hover-color` on hover, drag and as the drag proxy, and the FM's themes define those as `#f2f2f2` / `#5595F5` (neo-theme-neo-light) and `--gray-800` / `--purple-500` (neo-theme-neo-dark) — a blue or purple bar with a 5 px radius and 8 px margins, from a palette the cockpit does not use anywhere else.

The sites inside the cockpit root: `apps/agentos/view/fleet/memories/Container.mjs` (`fm-memories-splitter`, "the rail/reading split is the engine's Splitter"), `apps/agentos/view/fleet/activity/Container.mjs:358-362` (`fm-stream-splitter`, the observer mode's message beside the list), `apps/agentos/view/fleet/mailbox/Container.mjs:264-267` (`fm-mailbox-splitter`). The only FM rules on these classes hide them in the stacked layouts (`memories/Container.scss:374`, `activity/Container.scss:163`, `mailbox/Container.scss:107`).

## The Architectural Reality

- The content well is the ONE frame owner for hosted panes (`cockpit/Container.scss`, the `.neo-dashboard-dock-reveal-pane-slot` rule and its comment): a frame-level rule is where a pane-independent token mapping belongs, not three per-view copies.
- The engine splitter's contract is its token set (`--splitter-background-color`, `--splitter-border`, `--splitter-border-radius`, `--splitter-hover-color`, `--splitter-margin-horizontal`, `--splitter-margin-vertical`); the cockpit overrides tokens, never the engine selector's declarations.
- Build finding (2026-10-23): the FM declares its dock values on `.neo-dashboard-dock-splitter` itself, which is how they beat the engine's `.neo-dashboard` defaults — so pane content never inherits them; at the Memories pane `--dock-splitter-background` computes to the engine floor. The mapping therefore needs the dock-token rule's selector widened to `.neo-splitter` as well, beside the `.neo-splitter` token mapping.
- Build finding: the engine splitter's drag proxy mounts on `document.body`, outside the cockpit root; the mapping alone leaves a drag painting the theme color. The proxy gets its own body-level rule.
- Build finding: the Observatory (`ObservatoryContainer.scss:131-139`) is a keeper view under `.agent-shell`, outside `.fm-fleet-cockpit`; it already wears the dock's values through its own copy. Out of scope here; the copy remains a separate source of drift to fold when that view moves into the frame.

## The Fix

In `cockpit/Container.scss`, inside `.fm-fleet-cockpit`: the dock-token rule's selector widened from `.neo-dashboard-dock-splitter` to `.neo-dashboard-dock-splitter, .neo-splitter` (values unchanged), plus one `.neo-splitter` rule mapping the engine tokens onto the dock's (`--splitter-background-color: var(--dock-splitter-background)`, `--splitter-hover-color: var(--dock-splitter-background-hover)`, `--splitter-border: none`, `--splitter-border-radius: var(--dock-splitter-radius)`, margins 0 — the dock splitter's own radius and margins); and one body-level rule for `.neo-dragproxy.neo-splitter` carrying the dock's active color for the drag. Nothing per view; nothing in the engine.

Design authority: the dock splitter tokens are the cockpit's splitter language (`cockpit/Container.scss`, the `.neo-dashboard-dock-splitter` rule); the engine theme's splitter palette is not.

## Acceptance Criteria

- [ ] AC-1: on a source build, the Memories, Activity (observer mode) and Mailbox splitters compute the same rest and hover colors, radius and margins as a dock splitter in both skins, and the drag proxy paints the dock's active color mid-drag (computed-style reads in the visual suite; the Memories reading arm at 1280×1000 carries the golden).
- [ ] AC-2: no per-view splitter color rule is added; the engine's `Splitter.scss` and the Observatory's own rule are untouched.
- [ ] AC-3 (post-merge): the operator's installed app shows one splitter language in the Memories view and the main view.

## Out of Scope

The stacked-layout hiding rules (correct as they are); the dock splitter's own tokens; splitter hit-area or thickness (the pane splitters are the engine's 10 px against the dock's 6 px); the engine's 0.3 s hover transition versus the dock's motion token; the Observatory's own copy; a pane torn out into its own window (it leaves the cockpit root and wears the theme splitter again — inferred, a follow-up if it shows).

## Avoided Traps

Copying the color into each view's SCSS (drifts); styling `.neo-splitter` declarations directly instead of its tokens (fights the engine's hover/proxy rules); a mapping rule alone (the dock values are not inherited by pane content — build finding above).

## Related

#505 (parent: every important view renders correctly) · #506 / #715 (the Memories pane's reader and rail) · neomjs/neo `src/component/Splitter.mjs` + `resources/scss/src/component/Splitter.scss`.

Build finding, captured as a defect-note (not promoted): in a 1280×800 cockpit the Memories strip's grid body is 129.5 px, shorter than one 132 px row, so the engine's `updateAvailableRows` yields 0 rows and nothing renders while the meta line says "3 of 3 sessions".

handoff: Clio (design lead) drives it through a delegated builder; PR to follow.

Origin Session ID: d0801374-2558-4c56-97d0-ba4a719149a3

Handoff Retrieval Hints: `query_raw_memories("cockpit engine Splitter tokens dock splitter blue memories view drag proxy body")`; the operator's 2026-10-23 FM report.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session d0801374-2558-4c56-97d0-ba4a719149a3


## Timeline

- 2026-10-23T15:11:59Z @neo-fable-clio added the `bug` label
- 2026-10-23T15:11:59Z @neo-fable-clio added the `agent-os` label
- 2026-10-23T15:11:59Z @neo-fable-clio added the `ai` label
- 2026-10-23T15:11:59Z @neo-fable-clio added the `design` label
- 2026-10-23T15:13:36Z @neo-fable-clio added parent issue #505
- 2026-10-23T16:46:13Z @neo-fable-clio cross-referenced by #789
- 2026-10-23T17:04:04Z @neo-fable-clio cross-referenced by PR #792
- 2026-10-23T17:04:07Z @neo-fable-clio cross-referenced by PR #793
- 2026-10-23T17:24:21Z @neo-fable-clio cross-referenced by #794

