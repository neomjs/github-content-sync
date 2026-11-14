---
id: 963
title: Home's expired run reads as a link; the empty list names its date
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-11-14T03:49:20Z'
updatedAt: '2026-11-14T03:49:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/963'
author: neo-opus-ada
commentsCount: 0
parentIssue: 414
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
# Home's expired run reads as a link; the empty list names its date

## Context

Clio's design read of #959's three Look points, after #961 merged ([#961 comment 6871211471](https://github.com/neomjs/neo-agent-institution/pull/961#issuecomment-6871211471)), read from the Home baselines on `dev` (`home-operator-expired.png`, `-light.png`, `-nothing.png`). Two of the three points change what shipped. The third confirms it: the `expired` chip stays while its own list is in view, even at zero.

## The Problem

1. **The expired run is the loudest and the least link-like part of the line.** The operator's line reads `5 questions · 2 merges · 141 expired unanswered`. The run carries the line's own ink and no underline at rest. So "141" is the largest number on the line and draws the eye first, and it is the one link of three that does not look like one: the two counts before it are dotted, and it shows nothing until hovered.
2. **The expired list's empty line can be untrue.** The Mailbox's expired view says `no ask expired this week` when the list is empty. Its window starts at the producer's `since`, which is a week only when the window happens to be one.

## The Architectural Reality

- `resources/scss/src/apps/agentos/home/Container.scss`, the `&.is-expired` rule under the line's links, sets `text-decoration-line: none` and adds the underline only on `:hover`. The sibling count links carry the dotted underline at rest.
- `apps/agentos/view/fleet/mailbox/Container.mjs`, the `expired` entry of the view-copy map, has `empty: 'no ask expired this week'`, a fixed string. The expired read's answer already carries `since`, the window's start ISO, which Home's run uses for its title through `ViewerTime.formatViewerTime`.
- The chip at zero: `OperatorContainer.applyViewChips` keeps `expired · 0` visible while `view === 'expired'`. That is already as built, and this ticket leaves it alone.

## The Fix

1. In `Container.scss`, `&.is-expired` drops its `text-decoration-line` override, so the run takes the siblings' dotted underline at rest. It takes the secondary ink, `--fm-ink-dim`, if that ink measures at least 4.5 : 1 against Home's ground in both skins; otherwise the run keeps the line's ink and gains only the underline.
2. In the Mailbox, the expired view's empty line reads `no ask expired since <the answer's since, viewer date>`. An answer without `since` (an older plane) reads `no ask expired`, never a window it cannot name.

## Acceptance Criteria

- [ ] The expired run shows the dotted underline at rest, like the two counts before it, in both skins. The PR records the `--fm-ink-dim` contrast against Home's ground in each skin, and the ink applied follows from it.
- [ ] The expired view's empty line names the answer's `since` as a viewer date, and an answer without `since` names no window (unit arms for both).
- [ ] `home-operator-expired{,-light,-nothing}.png` and the Mailbox's expired-view frames are re-captured, with a BEFORE/AFTER pair in the PR body. `test-goldens` passes and the baselines are stamped.

## Out of Scope

- The chip at zero (as built and confirmed).
- The count's rule and its week-later read (#951).
- Any change to what the producer counts (Brain #1691).

## Related

Parent: #414. Source: #959 (closed by #961) and Clio's read above.

Ownership: self-assigned (`@neo-opus-ada`, #961's builder).

Live latest-open + closed sweep: the latest 20 open of 61 (red band, so filed under the owning epic #414) and a closed search ("expired asks Home underline"), read at 2026-11-14T03:50Z. The only same-surface hit is #959 itself. A2A sweep (`lane-claim`, latest 10): nothing on this surface. Memory Core: one query on the symptom, no prior ticket. Own assignments (#842, #516, #424): none on this surface.

Origin Session ID: 0f78c55e-2eb9-4b92-a44a-39ccc1af5a42
Retrieval Hint: "Home expired run dotted underline dim ink; expired list empty line names the window's since date"

## Timeline

- 2026-11-14T03:49:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-14T03:49:22Z @neo-opus-ada added the `enhancement` label
- 2026-11-14T03:49:22Z @neo-opus-ada added the `agent-os` label
- 2026-11-14T03:49:22Z @neo-opus-ada added the `ai` label
- 2026-11-14T03:49:22Z @neo-opus-ada added the `design` label
- 2026-11-14T03:49:27Z @neo-opus-ada added parent issue #414
- 2026-11-14T03:49:29Z @neo-opus-ada cross-referenced by PR #961
- 2026-11-14T04:00:42Z @neo-opus-ada cross-referenced by PR #964
- 2026-11-14T04:02:40Z @neo-opus-ada referenced in commit `d0720b2` - "test(agentos): drop the ticket refs from two test comments (#963)

Comment-only change; the re-stamp follows the visual spec's moved blob, no render changed."
- 2026-11-14T04:08:00Z @neo-opus-ada referenced in commit `334d95e` - "fix(agentos): the kind registry knows session-rotation events (#963)

Brain #1694 added session-rotation to FLEET_COCKPIT_EVENT_TYPES; the
registry's coverage arm derives from that list, so every Institution PR
run against Brain dev fails it, this one included. The same entry rides
#962 (7f0dd87); whichever merges first fixes dev, and the other's hunk is
identical."
- 2026-11-14T04:08:01Z @neo-opus-ada referenced in commit `2c3ce96` - "test(agentos): re-stamp after the kind registry entry (#963)"

