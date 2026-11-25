---
id: 1050
title: The Nightshift menu has no span shorter than 8 hours
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-24T23:56:37Z'
updatedAt: '2026-11-24T23:56:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1050'
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
# The Nightshift menu has no span shorter than 8 hours

Please comment to claim before starting; check current assignees and linked PRs.

## Context

The operator, 2026-11-24, after turning Nightshift on for the night: "minor friction item: menu options say 8h, 12h, until i get back. something like 4h or 6h would add value inside the dropdown."

The product bar's Nightshift toggle (#929, shipped in #933) offers three spans: "Until I switch it off", "For 8 hours" and "For 12 hours". An absence of a few hours has no fitting span. With 8 hours the fleet reads the operator as away for hours after they are back; with the open span someone has to remember to switch it off.

## The Problem

Nightshift tells peers whether an operator-only ask (merges, credentials, org admin) gets a quick answer; peers drive in both modes (`neomjs/neo-agent-brain#1625`). A span longer than the absence keeps the fleet on the slower path after the operator is back. The open span is the case the timed spans exist to avoid.

## The Architectural Reality

- `apps/agentos/view/NightshiftButton.mjs:10-14`: `NIGHTSHIFT_SPANS` is a frozen list of `{id, hours, text}`, and `hours: null` is the open span.
- `NightshiftButton.mjs:155-157`: the menu maps every span to one item, so a new span needs no menu code.
- `NightshiftButton.mjs:170-173`: `onSpanSelect(span, now)` fires `nightshiftchange` with `until = now + span.hours * 3_600_000`, so any whole-hour span works unchanged.
- `test/playwright/unit/apps/agentos/view/nightshiftButton.spec.mjs`, test "the span menu turns Nightshift on: open-ended, or ending after its hours", pins the menu's texts and checks every span that has hours.
- `test/playwright/e2e/agentos/NightshiftToggleNL.spec.mjs` clicks "For 8 hours" and stays valid.
- Design authority: n/a — this calls no existing behavior wrong. The spans' JSDoc already admits any span the plane enforces: "The ways to turn Nightshift on: until the operator switches it off, or for a span whose end the plane enforces as `until`, so a forgotten toggle cannot hold the fleet away."

## The Fix

Add two spans to `NIGHTSHIFT_SPANS`, between the open span and 8 hours: `{id: '4h', hours: 4, text: 'For 4 hours'}` and `{id: '6h', hours: 6, text: 'For 6 hours'}`. Update the unit spec's pinned list.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `NIGHTSHIFT_SPANS` (`apps/agentos/view/NightshiftButton.mjs:10`), the Nightshift menu | its JSDoc | five spans: open, 4, 6, 8 and 12 hours, in that order | none: an addition; existing ids and texts unchanged | the export's JSDoc, unchanged | the unit spec's pinned list |

## Acceptance Criteria

- [ ] The Nightshift menu lists, in this order: "Until I switch it off", "For 4 hours", "For 6 hours", "For 8 hours", "For 12 hours" (ids `open`, `4h`, `6h`, `8h`, `12h`).
- [ ] Choosing "For 4 hours" or "For 6 hours" sets `away` with an `until` 4 or 6 hours after the click. The unit spec's per-span check covers both once they are in the list.
- [ ] `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/apps/agentos/view/nightshiftButton.spec.mjs` passes, with the pinned list updated to the five texts.
- [ ] The PR body shows the open menu before and after.

## Out of Scope

- A custom duration or a time picker.
- Changing the open span, or the plane's expiry of `until` (`neomjs/neo-agent-brain#1625`).
- The silent-mode toggle for notifications (#995).
- Milestone FM v1: like #929, this is team tooling outside FM v1's journeys.

## Related

#929 (the toggle), #933 (its PR), #942 (its spacing).

Live latest-open + closed sweep: checked the latest 20 open (57 open, red band; no open owner for the span menu) and open and closed searches `Nightshift` (closed #929 with its Out of Scope read, and #942; open #995 is a different surface) at 2026-11-24T23:55Z; no equivalent or prior exclusion found.
A2A claim sweep: the 30 newest messages back to 22:46Z; no claim on this scope.
MC sweep: no prior decision on span lengths. Own-assignment sweep: my #995 covers notification silence, not spans.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: c17246cd-8314-4b49-96d8-0b79d46b445e
Retrieval Hint: "Nightshift span menu 4 hours 6 hours"

## Timeline

- 2026-11-24T23:56:38Z @neo-opus-ada added the `enhancement` label
- 2026-11-24T23:56:39Z @neo-opus-ada added the `good first issue` label
- 2026-11-24T23:56:39Z @neo-opus-ada added the `help wanted` label
- 2026-11-24T23:56:39Z @neo-opus-ada added the `agent-os` label
- 2026-11-24T23:56:39Z @neo-opus-ada added the `ai` label

