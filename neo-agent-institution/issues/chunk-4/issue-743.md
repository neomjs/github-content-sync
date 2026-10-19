---
id: 743
title: Awaiting-merge menu lacks structural FM token scope and link reset styles
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T13:06:19Z'
updatedAt: '2026-10-19T17:20:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/743'
author: neo-gemini-julia
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
closedAt: '2026-10-19T17:20:41Z'
---
# Awaiting-merge menu lacks structural FM token scope and link reset styles

Parent: #505.

## Context

Reported by `@tobiu` on the installed Fleet Manager (`2026-10-19`): opening the Fleet header's `N awaiting merge` popover (`AgentOS.view.fleet.roster.AwaitingMergeButton` → `AgentOS.view.fleet.roster.AwaitingMergeMenuList`) renders the pull-request rows (`#80 · test(grid): ...`, `#190 · docs(skills): ...`) as raw browser-default underlined blue/purple `<a>` links in stock system font, cramped flush against the popover border and focus outline with zero row padding.

## The Problem

Two concrete styling defects combine on `.fm-awaiting-merge-menu`:

1. **Missing structural FM token scope on `.fm-awaiting-merge-menu` (`resources/scss/src/apps/agentos/Viewport.scss:8–9`):**
   - `AwaitingMergeMenuList` (`apps/agentos/view/fleet/roster/AwaitingMergeMenuList.mjs:23`, `cls: ['fm-awaiting-merge-menu']`) is a framework-floating `Neo.menu.List` mounted directly under `document.body` (outside `.agent-os-viewport.neo-viewport`), sharing `AgentOS.util.FloatingMenuTheme` with the instance switcher menu (`apps/agentos/util/FloatingMenuTheme.mjs:9–10`: *"The instance switcher and the fleet head's awaiting-merge button share this one seam"*).
   - However, `resources/scss/src/apps/agentos/Viewport.scss:8–9` scopes the shell's structural FM tokens (`--fm-chip-*`, `--fm-text-display`, `--fm-text-body`, `--fm-text-detail`, `--fm-text-micro`, `--fm-text-chrome`, `--fm-space-1` through `--fm-space-4`) to `.agent-os-viewport.neo-viewport, .fm-instance-menu` and omits `.fm-awaiting-merge-menu`.
   - Because `.fm-awaiting-merge-menu` is a `document.body` child outside `.agent-os-viewport.neo-viewport`, every structural token consumed inside `resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss:41–106` (`--fm-space-1`, `--fm-space-2`, `--fm-text-body`, `--fm-text-micro`, `--fm-chip-radius`, `--fm-chip-pad-y`, `--fm-chip-pad-x`) resolves to `unset`:
     - `--menu-list-item-padding: var(--fm-space-2)` resolves to `0px` (rows and focus outlines sit flush against the menu border),
     - `padding: var(--fm-space-1)` on `.fm-awaiting-merge-menu` resolves to `0px`,
     - `gap: var(--fm-space-2)` on `.neo-list-item.fm-awaiting-merge-row` resolves to `0px`, and
     - `font: var(--fm-text-body)` on `.fm-awaiting-merge-ref` and `font: var(--fm-text-micro)` on `.fm-awaiting-merge-stale` / `.fm-stage-open` fail to resolve.

2. **Unstyled `<a href>` color and underline on `.fm-awaiting-merge-ref` (`resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss:70–78`):**
   - Each row in `AwaitingMergeMenuList#createItemContent` (`apps/agentos/view/fleet/roster/AwaitingMergeMenuList.mjs:89–96`) renders `<a class="fm-awaiting-merge-ref" href="https://github.com/..." target="_blank">`.
   - Browser user-agent rules (`a:-webkit-any-link { color: -webkit-link; text-decoration: underline; }`) apply directly to the `<a>` element and override the parent `.neo-list-item`'s `color: var(--menu-list-item-color)` (`var(--fm-ink-dim)`).
   - Unlike `.fm-detail-pr-ref` in `resources/scss/src/apps/agentos/fleet/detail/ParticipationContainer.scss`, `.fm-awaiting-merge-ref` in `AwaitingMergeButton.scss:70–78` sets neither `color` nor `text-decoration: none`.

## The Architectural Reality

- `apps/agentos/TOKENS.md:47`: mode-invariant structural tokens (`--fm-chip-*`, `--fm-text-*`, `--fm-space-*`, `--fm-motion-*`) land once at the shell root in `resources/scss/src/apps/agentos/Viewport.scss`, and floating `document.body` menus join that comma group to inherit the structural tokens without duplicating definitions, while direct viewport ground paint stays isolated in the `.agent-os-viewport.neo-viewport` block at `Viewport.scss:581`.
- `resources/scss/src/apps/agentos/fleet/instances/SwitcherButton.scss:50–97` and `resources/scss/src/apps/agentos/fleet/detail/ParticipationContainer.scss` establish the cockpit's menu-row and PR-link styling (`--fm-ink` / `--fm-ink-dim`, `text-decoration: none`, `--fm-space-2` row padding, `--fm-space-1` menu padding).

## The Fix

1. In `resources/scss/src/apps/agentos/Viewport.scss:8–9`, add `.fm-awaiting-merge-menu` to the `.agent-os-viewport.neo-viewport, .fm-instance-menu` structural-token selector group (leaving the direct-paint `.agent-os-viewport.neo-viewport` block at line 581 untouched).
2. In `resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss:60–78`, reset `<a class="fm-awaiting-merge-ref">` link presentation (`color: var(--fm-ink); text-decoration: none;` with hover/focus-visible clarity matching cockpit surfaces).
3. In `test/playwright/visual/FleetCockpitVisual.spec.mjs`, add computed-style assertions on `.fm-awaiting-merge-menu` and `.fm-awaiting-merge-ref` (verifying non-zero menu/row padding from `--fm-space-1`/`--fm-space-2`, `textDecorationLine: 'none'`, and token-resolved link color rather than browser UA link blue/purple) and refresh the three merge-queue visual baselines (`fleet-head-merge-queue-open.png`, `fleet-head-merge-queue-stage-enabled.png`, `fleet-head-merge-queue-stage-enabled-light.png`).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `resources/scss/src/apps/agentos/Viewport.scss:8–9` | `apps/agentos/TOKENS.md §04`, `FloatingMenuTheme.mjs:6–10` | `.fm-awaiting-merge-menu` joins `.agent-os-viewport.neo-viewport, .fm-instance-menu` so structural `--fm-space-*`, `--fm-text-*`, and `--fm-chip-*` tokens resolve on the `document.body`-mounted menu | Direct ground paint remains scoped to `.agent-os-viewport.neo-viewport` only (`Viewport.scss:581`) | Comment in `Viewport.scss` header | Computed-style assertion + visual baselines in `FleetCockpitVisual.spec.mjs` |
| `resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss:60–78` | `apps/agentos/TOKENS.md §04`, `SwitcherButton.scss:50–88`, `ParticipationContainer.scss` | `.fm-awaiting-merge-ref` sets `color: var(--fm-ink)` and `text-decoration: none` (no browser UA blue/purple underline); row/menu padding and gap resolve via `--fm-space-1` / `--fm-space-2` | Hover/focus-visible states preserve 4.5:1 contrast in both dark and light skins | SCSS comments | `FleetCockpitVisual.spec.mjs` assertions + dark/light snapshots |

Decision Record impact: none (aligns `.fm-awaiting-merge-menu` with `TOKENS.md §04` and the existing `.fm-instance-menu` floating-menu token contract).

## Acceptance Criteria

- [ ] AC-1: `.fm-awaiting-merge-menu` is included in the structural-token selector group in `resources/scss/src/apps/agentos/Viewport.scss:8–9` (and excluded from the direct-paint `.agent-os-viewport.neo-viewport` block at line 581), so `--fm-space-*`, `--fm-text-*`, and `--fm-chip-*` resolve on the floating menu in both `neo-theme-neo-dark` and `neo-theme-neo-light`.
- [ ] AC-2: `.fm-awaiting-merge-ref` in `resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss` sets `color: var(--fm-ink)` and `text-decoration: none`, eliminating raw browser link blue/purple and underline in both skins.
- [ ] AC-3: `test/playwright/visual/FleetCockpitVisual.spec.mjs` asserts computed padding, font family, `textDecorationLine: 'none'`, and token-resolved ink color on `.fm-awaiting-merge-menu` / `.fm-awaiting-merge-ref`, and updates `fleet-head-merge-queue-open.png`, `fleet-head-merge-queue-stage-enabled.png`, and `fleet-head-merge-queue-stage-enabled-light.png`.

## Out of Scope

- Changing `OpenPullRequest` model fields or `OpenWorkRead.mergeRows` data projection (`#512` / `#560`).
- Stage launch behavior in `AwaitingMergeMenuList` (`#696` / `#697`).

## Avoided Traps

- **Duplicating `--fm-space-*` / `--fm-text-*` / `--fm-chip-*` literals inside `AwaitingMergeButton.scss`:** `TOKENS.md §04` mandates that mode-invariant structural tokens live once in `Viewport.scss` and reach `document.body`-mounted floating menus via the root selector group.
- **Adding `.fm-awaiting-merge-menu` to the bottom `.agent-os-viewport.neo-viewport` direct-paint rule (`Viewport.scss:581`):** That block paints `background: var(--fm-ground)`, which would clobber the menu's `--fm-panel-2` overlay surface (`RA-1` on `#17544`).

## Related

- Parent: #505
- Follow-up to #449 (#483), #512 (#560), #696 (#697)

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` at `2026-10-19T13:01:00Z` (`sort: 'created'`, newest `#742`); no duplicate found.
A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at `2026-10-19T13:00:58Z`; no competing claim on `AwaitingMergeMenuList` or `AwaitingMergeButton.scss`.
Memory Core rationale sweep: `query_raw_memories` on `awaiting merge menu list PR view styling popover AwaitingMergeButton AwaitingMergeMenuList` returned no prior ticket on this styling defect.
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-institution`.
Structure map: N/A (`neomjs/neo-agent-institution`; modifies existing SCSS and visual spec files only).

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: "AwaitingMergeMenuList AwaitingMergeButton.scss Viewport.scss fm-awaiting-merge-menu fm-awaiting-merge-ref unstyled link underline zero padding"

Authored by Julia (Gemini 4 Argon, Antigravity) 🏛️

## Timeline

- 2026-10-19T13:06:20Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T13:06:21Z @neo-gemini-julia added the `bug` label
- 2026-10-19T13:06:21Z @neo-gemini-julia added the `agent-os` label
- 2026-10-19T13:06:22Z @neo-gemini-julia added the `ai` label
- 2026-10-19T13:06:22Z @neo-gemini-julia added the `design` label
- 2026-10-19T13:08:30Z @neo-gemini-julia added parent issue #505
- 2026-10-19T13:23:06Z @neo-gemini-julia cross-referenced by PR #745
- 2026-10-19T14:18:40Z @neo-gemini-julia referenced in commit `f858ecb` - "fix(fleet): add FM token scope and link reset to awaiting-merge menu (#743)"
- 2026-10-19T17:20:41Z @tobiu referenced in commit `f3bc589` - "fix(fleet): add FM token scope and link reset to awaiting-merge menu (#743) (#745)"
- 2026-10-19T17:20:41Z @tobiu closed this issue

