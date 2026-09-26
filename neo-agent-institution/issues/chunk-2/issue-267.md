---
id: 267
title: The keeper nav shows icons with tooltips; each right-rail item reads as its own chip
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T22:28:50Z'
updatedAt: '2026-09-26T22:48:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/267'
author: neo-opus-grace
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
---
# The keeper nav shows icons with tooltips; each right-rail item reads as its own chip

## Context

The operator's 2026-09-26 FM design corrections, recorded on #10 ([comment 5849980629](https://github.com/neomjs/neo-agent-institution/issues/10#issuecomment-5849980629)), include two navigation items. Design authority for both sits with me.

- **Right rail:** "Without hover, spaces inside labels and gaps between items look too similar to tell how many items exist." Each item must be visually distinguishable, as in `apps/workstation`.
- **Left navigation:** it is overloaded, and "icons with tooltips are the leading option to evaluate". This is not approval to regroup destinations.

## The Problem

- **Right rail:** the cockpit's right edge rail (Agent detail, Perspectives, Add agent, Wake routes) renders rotated labels on a transparent strip. The engine gives tabs a 2px gap and a transparent resting ground (`src/dashboard/Container.scss`, `--dock-rail-tab-background: transparent`), so the gap between two items reads like the word space inside "AGENT DETAIL".
- **Left navigation:** the keeper nav (`Viewport.mjs`, `agent-shell` tab bar) carries six rotated labels plus icons (Home, Fleet, Observatory, System, Accounts, Chat). The labels make the rail a column of text competing with the content.

## The Architectural Reality

- The rail tab's paint is engine capability, read through tokens (`--dock-rail-tab-background`, `-hover`, `-active`, `-color*`). The cockpit already sets hover and active values in `resources/scss/src/apps/agentos/fleet/cockpit/Container.scss`, so a resting value is one more token. The engine's `gap: 2px` stays.
- Keeper nav buttons are tab header buttons, which take a `tooltip` config like the theme switch in the same Viewport. A visually hidden label keeps the accessible name. `display: none` would drop it from the accessibility tree.

## The Fix

1. Right rail: give each tab a resting chip surface one tone above the ground (the cockpit's `--fm-rail`) and a small radius. Hover and active keep their current `--fm-panel` / signal-ink reading, and the 2px gap then separates chips, not words.
2. Left navigation: hide the keeper labels visually, keep them as the accessible name, and give each button a tooltip with its label. Icons stay as they are, and routes and order are unchanged.

## Acceptance Criteria

- [ ] AC-1: without hover, each right-rail item has its own visible surface, and adjacent items are separated by ground (visual golden, both skins).
- [ ] AC-2: the keeper nav shows icons only. Hovering a button shows its label as a tooltip, and the button's accessible name is still its label (unit or NL check on the rendered button).
- [ ] AC-3: the PR's re-recorded goldens show the nav and the rail before and after (GitHub's image diff), so the look is reviewable at the merge gate like any other change.

*AC-3 amended by the author: the first version made an extra operator approval a merge gate. The operator already merges every PR, and nav/rail design authority sits with me, so a second gate was deference, not a Tier-4 domain.*

## Out of Scope

- Regrouping or renaming destinations (#10's other suggestions); the health row (#246); the Chat destination's scope.

## Related

Parent: #10. Related: #246, #264.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T22:33Z hold no nav or rail ticket (#246 is the health row; #247 the strip head). A2A: Emmy's 21:19Z and 21:43Z relays name nav/rail authority as mine; no competing claim. Memory Core: the operator's corrections exist only as Emmy's #10 record; no prior nav/rail decision against this. Own assignments: #258, #264 (open PR #266); no overlap with these files, though #266's goldens and these share the stamp.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "keeper nav icons tooltips right rail chip distinguishable items"


## Timeline

- 2026-09-26T22:28:50Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T22:28:52Z @neo-opus-grace added the `enhancement` label
- 2026-09-26T22:28:52Z @neo-opus-grace added the `ai` label
- 2026-09-26T22:28:52Z @neo-opus-grace added the `design` label
- 2026-09-26T22:28:59Z @neo-opus-grace added parent issue #10
- 2026-09-26T22:45:54Z @neo-opus-grace cross-referenced by PR #268
- 2026-09-26T22:49:48Z @neo-opus-grace cross-referenced by #269

