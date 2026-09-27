---
id: 264
title: Remove the bottom Route graph pane; the Observatory keeps the route picture
state: CLOSED
labels:
  - enhancement
  - ai
  - design
  - refactoring
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T22:06:45Z'
updatedAt: '2026-09-27T09:31:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/264'
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
closedAt: '2026-09-27T09:31:09Z'
---
# Remove the bottom Route graph pane; the Observatory keeps the route picture

## Context

The operator's 2026-09-26 FM design ruling is recorded on #10 ([comment 5849980629](https://github.com/neomjs/neo-agent-institution/issues/10#issuecomment-5849980629)): "Bottom Route Graph must go away." The ruling covers that visible pane only, not the Golden Path producer, the text list or the Observatory. Emmy's collision check confirmed #258 does not touch the pane and handed the removal to me.

## The Problem

The cockpit's south strip (`stream-tabs`) carries two Golden Path surfaces: the text list (`goldenPath`) and the "Route graph" tab (`goldenPathGraph`), a 2D spine drawn by `AgentOS.canvas.GoldenPathGraph`. The Observatory (a left-rail view since `#243`) draws the same route in 3D and is the graph surface going forward (#258). The strip tab duplicates it, and it costs a canvas renderer, a layout util, a view pair, SCSS, goldens and specs.

## The Architectural Reality

- Declaration: `view/fleet/cockpit/Container.mjs` item `goldenPathGraph` (module `goldenpath/GraphContainer.mjs`, header "Route graph", reference `golden-path-graph`). `util/CockpitPerspectives.mjs#arrangement` lists it in `stream-tabs` for all three duties. The unit fixture `shippedDockDocument.mjs` mirrors both.
- Implementation: `view/fleet/goldenpath/GraphContainer.mjs` + `GraphCanvas.mjs`, `canvas/GoldenPathGraph.mjs`, `util/GoldenPathGraphLayout.mjs`, `resources/scss/src/apps/agentos/fleet/goldenpath/GraphContainer.scss`, `goldenPathGraphLayout.spec.mjs`, and the two Route graph tests in `FleetCockpitVisual.spec.mjs` with their goldens.
- Shared piece: `GoldenPathGraphLayout.describeCurrency` also feeds the Observatory's currency line (`goldenpath/ObservatoryContainer.mjs`). It moves beside `GoldenPathEnvelope.currency`, which already owns the currency reading, so the Observatory keeps it.
- Persistence: perspective captures key node and item ids (`PerspectiveLibrary`, `restoreSavedLayout`), so a capture saved before this change can still name `goldenPathGraph`.

## The Fix

1. Remove the item from the pane catalog and from `stream-tabs` in the declared perspectives, and from the fixture's mirror.
2. Delete the view pair, the renderer, the layout util, the SCSS and their specs and goldens. Move `describeCurrency` into `GoldenPathEnvelope` together with its arms.
3. A stored capture that still names the item restores without it: no error, no empty tab.

## Acceptance Criteria

- [ ] AC-1: the cockpit declares no Route graph item. The south strip in all three declared perspectives lists the six remaining tabs.
- [ ] AC-2: nothing imports `GraphContainer`, `GraphCanvas`, `GoldenPathGraph` or `GoldenPathGraphLayout`. The Observatory's currency line reads as before, pinned by a `describeCurrency` arm in the envelope spec plus the existing Observatory specs.
- [ ] AC-3: a persisted capture that names `goldenPathGraph` restores into the cockpit without error and without that tab.
- [ ] AC-4: the Route graph goldens are deleted. Any golden whose strip changed is re-recorded and reviewed, and the baseline stamp is refreshed.

## Out of Scope

- The Golden Path text list and its data producer; the Observatory's scene work (#258).
- The strip's shared head, inset and button scale (#247).

## Related

Parent: #10. Related: #258, #247, `#243`, `#230`.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T22:08Z hold no equivalent (newest #263). A2A in-flight sweep: Emmy's 21:23Z collision check and 21:43Z handoff name this removal as mine; no competing claim. Memory Core problem-noun sweep: Emmy's 21:30Z turn mapped the same file set and the shared `describeCurrency`; no prior decision against the removal. Own assignments: #258 shares the Observatory's currency helper, which this ticket keeps working, and #261.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "remove bottom Route graph pane GoldenPathGraph describeCurrency Observatory"

## Timeline

- 2026-09-26T22:06:46Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T22:06:47Z @neo-opus-grace added the `enhancement` label
- 2026-09-26T22:06:47Z @neo-opus-grace added the `ai` label
- 2026-09-26T22:06:47Z @neo-opus-grace added the `design` label
- 2026-09-26T22:06:47Z @neo-opus-grace added the `refactoring` label
- 2026-09-26T22:06:54Z @neo-opus-grace added parent issue #10
- 2026-09-26T22:25:42Z @neo-opus-grace cross-referenced by PR #266
- 2026-09-26T22:28:51Z @neo-opus-grace cross-referenced by #267
- 2026-09-26T22:40:14Z @tobiu referenced in commit `9c9a158` - "fix(agentos): a host without a pane catalog keeps every item when a capture applies (#264)

The retirement read Object.keys(me.panes), which throws on the projection
spec's partial host; that host has no catalog to retire against, so it now
applies the restored document unchanged. The visual stamp follows the
Container change; the goldens re-verified unchanged (17/17)."
- 2026-09-26T22:45:54Z @neo-opus-grace cross-referenced by PR #268
- 2026-09-27T08:16:32Z @tobiu referenced in commit `5897917` - "feat(agentos): the bottom Route graph pane leaves the cockpit, and a stored perspective naming it restores without it (#264)

The operator ruled the south strip's Route graph tab out (#10); the
Observatory draws the route going forward. The pane's catalog entry and
stream-tabs slot go, with its view pair, the 2D canvas renderer, its layout
util, its SCSS, its specs and its eight goldens.

describeCurrency moves beside GoldenPathEnvelope.currency, so the
Observatory's currency line reads as before (its arms move to the envelope
spec). A stored perspective that still names the retired pane, such as a
shared artifact exported before this change, restores without it:
CockpitPerspectives.retireUndeclaredItems closes it out with the engine's
closeItem semantics, so an emptied strip or split collapses."
- 2026-09-27T08:16:32Z @tobiu referenced in commit `d4d2863` - "test(visual): the cockpit goldens re-recorded without the Route graph tab (#264)

Seven cockpit goldens show the south strip, which lost its last tab; each
was re-recorded with --update-snapshots=all and compared against its
predecessor: the only change is the missing ROUTE GRAPH tab (and, at 314 px,
the strip's overflow that follows from it). accounts-config-surface and
system-view-cold also re-rendered with sub-tolerance drift unrelated to
this change and were left as they were. The input stamp is refreshed."
- 2026-09-27T08:16:32Z @tobiu referenced in commit `7df6e51` - "fix(agentos): a host without a pane catalog keeps every item when a capture applies (#264)

The retirement read Object.keys(me.panes), which throws on the projection
spec's partial host; that host has no catalog to retire against, so it now
applies the restored document unchanged. The visual stamp follows the
Container change; the goldens re-verified unchanged (17/17)."
- 2026-09-27T08:16:32Z @tobiu referenced in commit `d81d626` - "test(visual): re-stamp the baseline inputs on the rebased head (#264)"
- 2026-09-27T08:31:27Z @neo-opus-grace cross-referenced by #277
- 2026-09-27T08:41:38Z @tobiu referenced in commit `a2c11d3` - "feat(agentos): the bottom Route graph pane leaves the cockpit, and a stored perspective naming it restores without it (#264)

The operator ruled the south strip's Route graph tab out (#10); the
Observatory draws the route going forward. The pane's catalog entry and
stream-tabs slot go, with its view pair, the 2D canvas renderer, its layout
util, its SCSS, its specs and its eight goldens.

describeCurrency moves beside GoldenPathEnvelope.currency, so the
Observatory's currency line reads as before (its arms move to the envelope
spec). A stored perspective that still names the retired pane, such as a
shared artifact exported before this change, restores without it:
CockpitPerspectives.retireUndeclaredItems closes it out with the engine's
closeItem semantics, so an emptied strip or split collapses."
- 2026-09-27T08:41:38Z @tobiu referenced in commit `a66da68` - "test(visual): the cockpit goldens re-recorded without the Route graph tab (#264)

Seven cockpit goldens show the south strip, which lost its last tab; each
was re-recorded with --update-snapshots=all and compared against its
predecessor: the only change is the missing ROUTE GRAPH tab (and, at 314 px,
the strip's overflow that follows from it). accounts-config-surface and
system-view-cold also re-rendered with sub-tolerance drift unrelated to
this change and were left as they were. The input stamp is refreshed."
- 2026-09-27T08:41:38Z @tobiu referenced in commit `7499334` - "fix(agentos): a host without a pane catalog keeps every item when a capture applies (#264)

The retirement read Object.keys(me.panes), which throws on the projection
spec's partial host; that host has no catalog to retire against, so it now
applies the restored document unchanged. The visual stamp follows the
Container change; the goldens re-verified unchanged (17/17)."
- 2026-09-27T08:41:38Z @tobiu referenced in commit `c7e1fe3` - "test(visual): re-stamp the baseline inputs on the rebased head (#264)"
- 2026-09-27T08:46:46Z @neo-opus-grace cross-referenced by #278
- 2026-09-27T09:31:09Z @tobiu referenced in commit `03e6c43` - "Merge pull request #266 from neomjs/grace/264-remove-route-graph

feat(agentos): the bottom Route graph pane leaves the cockpit, and a stored perspective naming it restores without it (#264)"
- 2026-09-27T09:31:09Z @tobiu closed this issue

