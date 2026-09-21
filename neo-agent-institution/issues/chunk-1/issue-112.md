---
id: 112
title: Roster card names a seat active without a presence beacon
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T22:35:30Z'
updatedAt: '2026-09-12T16:59:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/112'
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
closedAt: '2026-09-12T16:59:04Z'
---
# Roster card names a seat active without a presence beacon

## Context

On 2026-09-04 the Clio seat had run with no projected hooks for days (the class Brain #317 fixes): `who_is_online` graded it `idle` from `add_memory` recency with no turn-presence beacon, and the roster showed a normal band. The operator's direction the same evening: seat-hook health is a Fleet Manager concern. The FM cannot reach a seat's filesystem; what it can do is name the effect's absence — an active seat whose hooks never beacon.

## The Problem

The card projects the graded band only (`apps/agentos/view/fleet/roster/card/Container.mjs:410-416`, `PRESENCE_BAND_LABEL`); the presence axis is a passthrough `{source, state, confidence, lastSeenAt, reason?, validationState?, since?}` (`apps/agentos/model/FleetAgent.mjs:86-94`), and nothing on it says whether the band came from a beacon or from write recency. A seat with dead hooks and a seat with live hooks read the same.

## The Architectural Reality

The Brain adapter computes the distinction and drops it; the producer leaf (Brain: "The presence row says whether a seat's beacon is fresh, stale or absent") adds `beacon: fresh | stale | absent | unobserved` to the row. The FM never re-derives presence facts (Brain #53's contract: three independent signals, none inferring another); it words a fact the row carries.

## The Fix

`FleetAgent.presence` passes `beacon` through (no coercion). The card renders one word beside the band, from the token set already loaded: `beacon absent` in the state-idle amber when `beacon === 'absent'` and the band is `online` / `idle` (active by writes, silent hooks); `beacon stale` in ink-dim when `stale`; nothing when `fresh`; nothing when `unobserved` (the presence-capability chip already says the read failed). The band's glyph carries the facet at every card width (`◌` absent · `◎` stale, the diagnostic on the band's title and aria-label); the word is a chip in the card's existing chip row that renders only where the state line holds it whole — no new color, no new type role, no layout change at the 240px band.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetAgent.presence.beacon` | the Brain presence row (producer leaf) | passthrough, closed enum | field absent (older Brain) → no word | model docblock | DTO-to-record arm |
| presence band + card chip | the band's own exception shape (glyph · emphasis · title/aria) + the existing chip idiom with the state-idle token | the band's glyph marks `absent` (active band) and `stale` at every width; one word beside it where the state line holds the widest ordinary line plus it (≥ 340px line) | none for `fresh` / `unobserved` | card JSDoc | unit arms per value; the 314px golden unchanged for `fresh`; a geometry arm across the card-width matrix |

## Acceptance Criteria

- [ ] A roster row with `beacon: 'absent'` and band `online` / `idle` hollows the band's ring (`◌`) at every width and renders `beacon absent` in the card's chip row where the state line holds it (≥ 340px line); `stale` renders `◎` and `beacon stale`; `fresh` and `unobserved` render no mark and no chip; a row without the field renders neither (older Brain).
- [ ] The band label and every other card fact are unchanged; the 314px synthesis goldens stay byte-identical for the `fresh` case.
- [ ] The word never appears with a `dark` / `benched` / `neverConnected` band — an inactive seat has no hook to be silent.
- [ ] Unit arms per value; one pipeline arm from a fixture DTO row to the rendered chip.

## Out of Scope

Re-projecting hooks from the FM (the projector runs on the seat host; Brain #317); the System view (#21); a second presence color.

## Related

Sub of #10. Blocked by the Brain producer leaf (cross-repo; named by title until the wire carries the field). Brain #53 (presence contract), #317, #79. Live latest-open sweep: latest 20 open Institution issues at 2026-09-04 ~22:25Z, no equivalent (closed #35 is the validation-provenance sibling); A2A last-30 all-states at 22:28Z, no claim; Memory Core recall: the unscoped query drowned in boot noise — null, not clean.

Origin Session ID: 49133900-1f86-4134-a82b-30ff0709bcaf
Retrieval Hint: "roster card beacon absent while active silent hooks presence chip"


## Timeline

- 2026-09-04T22:35:30Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T22:35:31Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T22:35:31Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T22:35:32Z @neo-fable-clio added the `ai` label
- 2026-09-04T22:35:32Z @neo-fable-clio added the `design` label
- 2026-09-04T23:31:49Z @neo-fable-clio cross-referenced by #113
- 2026-09-05T12:32:10Z @neo-fable-clio cross-referenced by #115
- 2026-09-05T15:31:38Z @neo-fable-clio referenced in commit `3218f84` - "test(fleet): a touched card-spec comment names the behavior, not a review round (#112)"
- 2026-09-05T15:32:36Z @neo-fable-clio referenced in commit `f57edad` - "test(fleet): the touched card comments name the behavior, not a review round or a ticket (#112)"
- 2026-09-05T15:32:37Z @neo-fable-clio cross-referenced by PR #119
- 2026-09-05T15:33:40Z @neo-fable-clio referenced in commit `e8a1e48` - "test(fleet): the touched card comments name the behavior, not a review round or a ticket (#112)"
- 2026-09-05T15:37:38Z @neo-fable-clio referenced in commit `abffe49` - "merge(dev): the tasks pane landed beside the beacon word — one visual stamp over both (#112)"
- 2026-09-12T09:57:43Z @neo-fable-clio referenced in commit `43e267b` - "fix(fleet): the beacon words yield to the state line, and the band's glyph carries the facet at every width (#112)

The state line is a nowrap row whose regular members already fill it below ~340px
(measured: dot · the longest state word · the longest active band · a two-digit badge =
247px; the words add 86px), so a fifth text member at every width overlapped the verbs at
the narrow band and wrapped a clipped row at the sub-narrow band.

The presence band now carries the facet at every width: its glyph hollows out (◌ absent,
◎ stale), absent takes the validation exception's emphasis pair, and the band's title and
aria-label speak the diagnostic. The words stay a sibling chip, presentational (aria-hidden),
rendered only where the line holds the widest ordinary line plus them — the state line is its
own named width-query context, and the words hide below a 340px line.

Coverage: the unit arms assert the band's glyph, classes, aria-label and title per facet
(both exceptions at once included) beside the words; a geometry arm across the card-width
matrix (240 · 314 · 320 · 410 · 498 · 500 · 720) pairs every beacon row with a control and
asserts the glyph at every width, the words exactly where the line holds them (inside the
line, before the badge, never under the verbs), and no clip or line-height delta against the
control — red on the pre-fix card at the glyph, red on the pre-fix SCSS at the yield."
- 2026-09-12T10:03:01Z @neo-fable-clio referenced in commit `1356338` - "chore(fleet): the render spec's comments name behavior, not review rounds (#112)"
- 2026-09-12T10:04:58Z @neo-fable-clio cross-referenced by #123
- 2026-09-12T13:17:37Z @neo-fable-clio cross-referenced by #128
- 2026-09-12T16:32:20Z @neo-fable-clio referenced in commit `34b935b` - "feat(fleet): the roster card names a seat active without a presence beacon (#112)

One chip word beside the presence band from the producer's turn-presence facet: 'beacon absent' when an active band's row carries no beacon (the hooks never beaconed while add_memory kept the seat green — invisible on every band until now), 'beacon stale' when the beacon went past its horizon. Nothing for fresh or unobserved, nothing for a row without the field (an older Brain), never on dark, benched or never-connected. The band label and every other card fact are unchanged; the goldens are byte-identical."
- 2026-09-12T16:32:20Z @neo-fable-clio referenced in commit `17b9ebf` - "test(fleet): the touched card comments name the behavior, not a review round or a ticket (#112)"
- 2026-09-12T16:32:20Z @neo-fable-clio referenced in commit `ae2cd54` - "fix(fleet): the beacon words yield to the state line, and the band's glyph carries the facet at every width (#112)

The state line is a nowrap row whose regular members already fill it below ~340px
(measured: dot · the longest state word · the longest active band · a two-digit badge =
247px; the words add 86px), so a fifth text member at every width overlapped the verbs at
the narrow band and wrapped a clipped row at the sub-narrow band.

The presence band now carries the facet at every width: its glyph hollows out (◌ absent,
◎ stale), absent takes the validation exception's emphasis pair, and the band's title and
aria-label speak the diagnostic. The words stay a sibling chip, presentational (aria-hidden),
rendered only where the line holds the widest ordinary line plus them — the state line is its
own named width-query context, and the words hide below a 340px line.

Coverage: the unit arms assert the band's glyph, classes, aria-label and title per facet
(both exceptions at once included) beside the words; a geometry arm across the card-width
matrix (240 · 314 · 320 · 410 · 498 · 500 · 720) pairs every beacon row with a control and
asserts the glyph at every width, the words exactly where the line holds them (inside the
line, before the badge, never under the verbs), and no clip or line-height delta against the
control — red on the pre-fix card at the glyph, red on the pre-fix SCSS at the yield."
- 2026-09-12T16:32:20Z @neo-fable-clio referenced in commit `f255e0e` - "chore(fleet): the render spec's comments name behavior, not review rounds (#112)"
- 2026-09-12T16:59:04Z @tobiu referenced in commit `fe898a9` - "Merge pull request #119 from neomjs/agent/112-beacon-card-word

feat(fleet): the roster card names a seat active without a presence beacon (#112)"
- 2026-09-12T16:59:05Z @tobiu closed this issue

