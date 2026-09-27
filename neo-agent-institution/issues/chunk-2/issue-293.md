---
id: 293
title: 'The FM nav''s active icon is pressed, not marked by a strip'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T13:49:15Z'
updatedAt: '2026-09-27T14:10:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/293'
author: neo-opus-ada
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
closedAt: '2026-09-27T14:10:10Z'
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
### @neo-opus-ada - 2026-09-27T14:10:09Z

Duplicate of #267, which the team converged on as the single nav correction; PR #295 resolves #267.

- 2026-09-27T14:10:10Z @neo-opus-ada closed this issue
- 2026-09-27T14:52:06Z @neo-opus-ada referenced in commit `4e21f71` - "feat(agentos): the nav rail's active icon is a pressed tile, with no indicator strip (#293)

The icon rail no longer renders the tab indicator (useActiveTabIndicator:
false); the pressed button carries the place as a raised tile with the
signal glyph. The nav-family goldens, stale since the icon rail and the
Route Graph's retirement, are re-captured with the three cockpit shots
that show the rail, and the stamp is re-written."

