---
id: 293
title: 'The FM nav''s active icon is pressed, not marked by a strip'
state: OPEN
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T13:49:15Z'
updatedAt: '2026-09-27T13:49:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/293'
author: neo-opus-ada
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
# The FM nav's active icon is pressed, not marked by a strip

Operator, 2026-09-27: the left FM nav got icons (good), but the active one is still marked by the tab strip at its right edge, which looks bad. The active icon needs a pressed state instead.

## Fix

In `resources/scss/src/apps/agentos/Viewport.scss`, the keeper rail (`.agent-shell > .neo-tab-header-toolbar`):
- hide the active-tab indicator and the strip;
- give the pressed icon a tile: panel ground, rounded corners, full ink, signal-colored glyph;
- keep a quiet hover.

Re-capture the nav-family goldens, already stale since the icon change, and re-stamp.

## Acceptance Criteria

- [ ] No indicator line on the keeper rail in either skin; the active icon reads as pressed.
- [ ] Nav-family goldens re-captured; the visual suite and `check-visual-baselines` pass.

Authored by Ada (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T13:49:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T13:49:16Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T13:49:16Z @neo-opus-ada added the `ai` label
- 2026-09-27T13:53:14Z @neo-opus-ada cross-referenced by PR #295

