---
id: 69
title: Empty-fleet CTA sits at the pane bottom and renders its label nearly black
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-01T21:40:04Z'
updatedAt: '2026-09-01T23:02:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/69'
author: neo-fable-clio
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
closedAt: '2026-09-01T23:02:59Z'
---
# Empty-fleet CTA sits at the pane bottom and renders its label nearly black

## Context

Operator-observed on the live cockpit (2026-09-01, connected Brain, empty fleet): the roster's bootstrap CTA "Add your first agent" renders low in the roster pane — visually centered on a box that seems to include the south strip — and its label and plus glyph are far darker than every other button's text. Both are defects of one component; the operator asked whether either was intentional. Neither is: the design ruling on record (`apps/agentos/view/fleet/roster/Container.mjs`, the `items` docblock) wants "a real button in the card region", and its SCSS declares `color: var(--fm-ink)`.

## The Problem

The empty fleet is the first-run moment of the product: the only thing on the roster is the path to the first agent, and it currently reads as an afterthought parked at the bottom in a colour that does not belong to the family. First paint is the design gate's floor (#13); this surface fails it before any interaction.

## The Architectural Reality

- Placement (code read): the CTA is the roster container's LAST vbox item, after the `RosterList` (`flex: 1`). With count 0 the list keeps the whole remaining height and the CTA lands below it; `align-self: center` + `margin: 24px auto` (`resources/scss/src/apps/agentos/fleet/roster/Container.scss`, `.fm-fleet-empty-cta`) center it horizontally only.
- Colour (read in the lane): the Engine paints `.neo-button-text` / `.neo-button-glyph` from `--button-text-color` / `--button-glyph-color`, each with `-hover` and `-active` siblings, and `.neo-button:hover` / `:active` (0,2,0) re-paint the plate and border from `--button-background-color-*` / `--button-border-*`; the themes' defaults are the reversed ink and the primary surface of a filled button. A `color`, `background` or `border` on the button element never reaches the spans and loses on every interaction state — the token channel is the only channel.
- The CTA renders only at roster count 0 (`hidden` toggled by the count), so the witness needs an empty roster: the authenticated e2e harness with a `fleetRoster` fixture of zero rows.

## The Fix

- The empty state owns the card region: the list and the CTA share one box (the list stays mounted as the scroll owner; the CTA centers over it) so the button centers between the controls row and the pane's bottom edge — a placement, not a specificity bump.
- The label, glyph, plate and border carry the family's ink and plate through the button's own token channel in every interaction state the Engine gives the button (rest, hover, press), not a `!important` or a deeper selector.
- A Neural Link witness boots the cockpit against a zero-row roster fixture and reads the CTA's rect against the card region's rect and the computed colour of the label and glyph at rest, under hover and under press; the visual goldens are re-captured only if their pixels move.

## Acceptance Criteria

- [ ] Against an empty roster the CTA's vertical center is the card region's vertical center (DOM-read receipt, before/after numbers in the PR).
- [ ] The CTA's label and glyph compute to the family's button ink in both themes — at rest, under hover and under press — and the dashed plate survives both interaction states.
- [ ] A populated roster shows no CTA and no layout change (the existing roster goldens stay byte-stable).
- [ ] The witness lives beside the Neural Link battery (`test/playwright/e2e/agentos/`, collected only under the `NEO_AGENTOS_RUNTIME_ROOT` binding — Institution CI never executes a Neural Link witness; the `npm run test-e2e:nl` script arrives with #70) and is green locally, red-first; the CI unit/visual suites stay green.

## Out of Scope

- The activity row alignment and kind-chip width family — #63 (same design pass, its own lane).
- New empty-state copy or a new illustration; the ruling's button stays the button.

## Related

Parent: #10. Sibling: #63. Design-gate epic: #13.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-01T21:40Z (nearest: #63, the kind-chip width family — different surface); A2A claims in the last hour cover Engine popup drag, skills 0.1.3 and the wake manifest — no overlap; Memory Core raw query on the CTA returned nothing.

Amended 2026-09-01T22:45Z after PR #72's review round: the colour reality names the full token channel (rest / hover / press), AC-2 is state-qualified, and AC-4 no longer names a script this repository does not carry yet.

Origin Session ID: f353cda0-1b36-49e4-9b73-3f7aec1896e0

Retrieval Hint: `roster empty state CTA add your first agent placement colour card region`

📜 Clio


## Timeline

- 2026-09-01T21:40:04Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-01T21:40:06Z @neo-fable-clio added the `bug` label
- 2026-09-01T21:40:06Z @neo-fable-clio added the `agent-os` label
- 2026-09-01T21:40:06Z @neo-fable-clio added the `ai` label
- 2026-09-01T21:40:06Z @neo-fable-clio added the `design` label
- 2026-09-01T21:51:30Z @neo-fable-clio cross-referenced by PR #71
- 2026-09-01T21:59:25Z @neo-fable-clio cross-referenced by PR #72
- 2026-09-01T22:34:51Z @neo-fable-clio referenced in commit `45543e7` - "chore(test): merge dev and refresh the visual baseline stamp (#69)"
- 2026-09-01T22:45:21Z @neo-fable-clio referenced in commit `c1162f7` - "fix(agentos): the empty-fleet CTA keeps its plate and ink under hover and press (#69)

The first cut set five text/glyph tokens and kept element-level background and border; the Engine's .neo-button:hover / :active (0,2,0) re-paint plate, border, label and glyph from the -hover / -active tokens, so the glyph fell back to the theme's reversed ink under press (the missing --button-glyph-color-active) and the dashed plate gave way to the theme's primary surface and border: none on touch. The CTA now carries the full channel — background, border, text and glyph in all three states, plus background-image and border-radius — and no element-level paint. The witness reads label, glyph, plate and border under hover and under press in both themes; red-first on the old tokens (the dashed plate lost under hover)."
- 2026-09-01T22:51:59Z @neo-fable-clio referenced in commit `3f5a730` - "chore(test): merge dev after #70 and refresh the visual baseline stamp (#69)"
- 2026-09-01T23:02:59Z @tobiu closed this issue
- 2026-09-01T23:02:59Z @tobiu referenced in commit `cc6ca55` - "Merge pull request #72 from neomjs/agent/69-empty-fleet-cta

fix(agentos): the empty fleet's CTA centers in the card region and wears the family ink (#69)"

