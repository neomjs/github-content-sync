---
id: 128
title: 'Roster cards reserve 126 px for rows no live row fills, so the roster reads as headers floating in dark slabs'
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T13:17:36Z'
updatedAt: '2026-09-18T15:08:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/128'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-18T15:08:44Z'
---
# Roster cards reserve 126 px for rows no live row fills, so the roster reads as headers floating in dark slabs

## Context

Dogfooding on the live fleet, 2026-09-12 (the operator, a visitor, the Neural Link): "gaps between fleet roster cards now look like cards". Measured on the live cockpit (9 agents, three columns at 1451 px): every card is 436×126 — the list's fixed `itemHeight: 126` — while its content is 78 px: identity row 40 px, state line 30 px, 8 px between; the remaining 48 px is card surface with nothing in it. Every live row today is `external harness · wake off` with one state line; the lane and activity rows the card was composed for are `not wired` for un-managed seats, so nothing ever fills the reserve. Card surface `--fm-panel` (#141a23) over the pane ground is a low-contrast pair, so the eye reads the identity rows as the content and the empty lower halves plus the 10 px gaps as one dark field: the cards look like gaps and the gaps like cards. The checked-in golden (`cockpit-default-shell.png`) shows the same shape on the static roster — the standing design at a density the design never met, not an engine-pin regression.

## The Problem

The card's height is a layout constant (the animate plugin's contract requires one fixed `itemHeight` per list), chosen for the composition that carries lane and activity rows. At live density with un-managed seats the reserved rows are empty on every card, so the roster's rhythm is 40 px of content per 136 px of pitch.

## The Architectural Reality

- `AgentOS.view.fleet.roster.List`: `itemHeight: 126`, `pluginAnimateConfig: {minItemWidth: 410}`; `Neo.list.plugin.Animate` positions the `li` slots absolutely from `itemHeight` × columns (`Animate.mjs`, `rows = floor(height / itemHeight)`); the card fills its slot (`> .fm-agent-card { height: 100% }`, `roster/Container.scss`).
- Card composition: `roster/card/Container.mjs` + `Container.scss` (`--fm-panel` surface, 1 px `--fm-line-soft` border, 9 px radius, padding 12/16). The §04 rhythm record in `roster/Container.scss` already defers "off-rhythm survivors" to a density pass.
- The July re-baseline picked this composition (neomjs/neo#14805, neomjs/neo#15536 — three directions, the operator's pick) at a density that carried lane rows.

## The Fix (design direction, then implement)

Options, each with a falsifier:

- (a) **Content-height cards:** `itemHeight` = the one-line composition (≈ 80 px) while the lane/activity rows are unwired; grow it when a second row ships (one number per list — the row count is list-level state, not per card). Falsifier: the verb cluster at the narrow bands (#123) still fits.
- (b) **Fill the reserve honestly:** render the lane row with its honest `no current lane reported` line on every card. Falsifier: nine identical grey lines add noise, not information.
- (c) **Separate cards from ground:** lift the card surface one step (`--fm-panel-2`) or strengthen the border so the slab reads as a card even when empty. Falsifier: the goldens re-render; the hierarchy against the selected and hover states holds.

Recommendation: (a) with (c); (b) rejected. The direction is the operator's call (design authority); the implementation is this lane's.

## Acceptance Criteria

- [ ] The roster's pitch follows its cards: no fixed row constant, every card takes the tallest card's measured height (operator ruling 2026-09-18: cards in a grid have the same height — this criterion's first wording, "no reserve taller than the padding", predates it). The 30 px work-row reserve on unobserved seats stays unless the operator rules otherwise: without it the whole roster re-pitches the moment the first agent starts working.
- [ ] Cards read as cards against the pane ground at the golden's viewport and at two-column width; goldens re-captured deliberately, by-eye diff in the PR.
- [ ] The animate plugin's `itemHeight` contract stays honored (one number per list); FLIP motion unchanged; the `AgentCardSynthesisRenderNL` matrix green.

## Out of Scope

The narrow-band overflow (#123); the beacon and presence semantics (#112).

## Related

#10 (parent epic); #123; #112; neomjs/neo#14805, neomjs/neo#15536.

Live latest-open sweep: the latest 20 open issues of neomjs/neo-agent-institution at 2026-09-12T13:14Z — none equivalent. A2A in-flight claim sweep: the last 30 messages carry no claim on this scope. Memory Core rationale sweep: the July re-baseline picked the composition; no decision covers live density. Own-assignment sweep: #123, #112 (same card, different defects).

Origin Session ID: fcdd7d71-e7bd-46e8-bd01-5d46ea620205
Retrieval Hint: "roster card itemHeight 126 density empty reserve"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session fcdd7d71-e7bd-46e8-bd01-5d46ea620205


Related, added 2026-09-12 after the operator challenged the fixed itemHeight: neomjs/neo#18604 — the animate plugin measures its row height from the tallest item (opt-in, fixed mode unchanged); option (a) above depends on it and lands with the pin that carries it.



## Timeline

- 2026-09-12T13:17:36Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-12T13:17:37Z @neo-fable-clio added the `enhancement` label
- 2026-09-12T13:17:38Z @neo-fable-clio added the `ai` label
- 2026-09-12T13:17:38Z @neo-fable-clio added the `design` label
- 2026-09-12T13:25:45Z @neo-fable-clio cross-referenced by #18604
- 2026-09-12T16:48:04Z @neo-fable-clio cross-referenced by #131
- 2026-09-12T17:09:33Z @neo-fable-clio cross-referenced by PR #132
- 2026-09-12T21:01:34Z @neo-fable-clio cross-referenced by #133
- 2026-09-13T15:53:13Z @neo-fable-clio cross-referenced by PR #18670
- 2026-09-13T19:10:55Z @neo-fable-clio cross-referenced by #136
- 2026-09-14T00:40:14Z @neo-fable-clio cross-referenced by PR #137
- 2026-09-14T01:11:30Z @neo-gpt cross-referenced by #10
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149
- 2026-09-18T10:40:56Z @neo-fable-clio cross-referenced by #151
- 2026-09-18T12:36:38Z @neo-fable-clio cross-referenced by #18874
### @neo-fable-clio - 2026-09-18T12:37:01Z

**Design ruling + dependency (2026-09-18).** Tried option (a) through the engine's measured mode (neo#18604) on a branch: the calm roster tightens as intended (126 → 104 px: head + work row, no dead reserve), but with mixed card anatomy the pitch follows the tallest card in the whole list while the other cards stay content-height — short cards float in tall slots (the `AgentCardSynthesisRenderNL` matrix shows it plainly), and one card showing a control-status re-pitches every row. Shown to @tobiu with before/after captures; his ruling: **cards in a grid have the same height.**

So the branch was dropped unshipped, and option (a) now means: measured rows **with** equal item heights. That needs the engine to size items to the measured row without falling into the grow-only trap — filed as neo#18874 (mine). This ticket is blocked on it; the roster change itself stays the two config lines (`itemHeight: null`, `pluginAnimateConfig.measureItemHeight: true`), and the goldens re-render once, from a full visual run, with equal cards in both the calm and the pathological fixture.

Also measured on the way: the remaining emptiness in the offline goldens is the work row of unobserved seats (`.fm-card-work-row {min-height: 30px}`), which live data fills — it is anatomy, not reserve, and collapsing it would make the cards unequal again. Option (c) (lifting the card surface) reads as unnecessary once the dead reserve is gone; still the operator's call after the first equal-height capture.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59

- 2026-09-18T13:03:35Z @neo-fable-clio cross-referenced by #153
- 2026-09-18T13:34:22Z @neo-opus-grace cross-referenced by PR #18886
- 2026-09-18T13:40:14Z @neo-fable-clio cross-referenced by #155
- 2026-09-18T14:17:27Z @neo-fable-clio cross-referenced by #157
- 2026-09-18T14:29:51Z @neo-fable-clio cross-referenced by PR #158
- 2026-09-18T14:50:35Z @neo-fable-clio cross-referenced by PR #159
- 2026-09-18T15:08:44Z @tobiu referenced in commit `a94bbd1` - "Merge pull request #159 from neomjs/agent/128-roster-measured-rows

feat(roster): the roster's cards take one measured height instead of a fixed 126px row (#128)"
- 2026-09-18T15:08:45Z @tobiu closed this issue
- 2026-09-18T17:04:02Z @neo-fable-clio cross-referenced by #123
- 2026-09-19T12:53:40Z @neo-gpt-emmy cross-referenced by PR #169

