---
id: 123
title: 'Roster card: the state line runs under the verbs — it needs a fit policy, not only at the narrow bands'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-12T10:04:57Z'
updatedAt: '2026-09-19T12:57:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/123'
author: neo-fable-clio
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
closedAt: '2026-09-19T12:57:18Z'
---
# Roster card: the state line runs under the verbs — it needs a fit policy, not only at the narrow bands

## Context

The roster card's state line — dot · state word · presence band · beacon words · lane-count badge (right-pinned) · telltale — is a nowrap hbox whose members are all `flex: none`. It does not fit its own ordinary content, and not only at the narrow bands: members run past the identity column and under the lifecycle verbs.

**Re-measured at intake, 2026-09-18, on dev@d3f17ec** (after #128's measured rows and #112's final form), under the Neural Link with the card-width matrix (`AgentCardSynthesisRenderNL.spec.mjs`'s pinning path). Cells: px past the line's right edge; **bold** = the member sits under the verbs.

| card | line | `working · ◉ fresh · 1 lane` | `rate-limited · ◉ active turn · 23 lanes` | shortest line + `throttle overage` | widest line + both telltale axes |
|---|---|---|---|---|---|
| 294 (narrow, 44px verbs) | 118 | **badge +50** | **band +57 · badge +129** | **telltale +179** | +390 |
| 319 (narrow) | 143 | **badge +25** | **band +32 · badge +104** | **+154** | +365 |
| 320 (regular) | 160 | badge +8 | **band +15 · badge +87** | **+137** | +348 |
| 410 (the roster's column floor) | 250 | fits | fits (246.8) | **telltale +47** | **+258** |
| 500 | 340 | fits | fits | fits | **+168** |
| 240 (sub-narrow, wraps) | 178 | 1 row | 2 rows | 2 rows | 3 rows, telltale 75px past the card |

The original table (2026-09-12) still holds cell for cell in the narrow band (50 · 30 · 25px). Three findings go past it:

1. **The telltale overflows at the roster's ordinary widths.** At the 410px column floor a single-axis chip on the SHORTEST line ends 47px past the line, under the verbs. Both axes (`wake suppressed · throttle rate-limited`, 252.8px) need a 508px line — a 668px card.
2. **The state word and the band alone exceed the narrow line:** `rate-limited` + `◉ active turn` = 174.6px against 95–143px. Marks for the badge and the telltale cannot meet the first acceptance criterion, whatever their thresholds. The closed vocabularies are also wider than the first pricing assumed: `benched / offline` 102px, `◉ never connected` 102px, the band's ` · validation stale` +114px.
3. **Since #128 the sub-narrow wrap no longer clips, it re-pitches:** cards share one measured height, so one dense card makes EVERY card 177–191px instead of 143.

Independently observed by @neo-gpt while reviewing PR #119 ("the fresh control already has lane-badge spill").

## The Problem

The card's own width modes (ADR 0029) size the avatar, the verbs and the engine tag per band, but the state line has no fit policy: every member keeps its full text and the line overflows — under the verbs at the narrow and regular bands, into extra rows at the sub-narrow band. The vocabularies vary too much for width thresholds alone (words 24–102px, bands 36–217px): priced for the worst case they hide facts that would fit; priced for the ordinary case the long ones still overflow. The visual 314 arm guards only the STATE WORD, so every other member fails silently.

## The Architectural Reality

- `resources/scss/src/apps/agentos/fleet/roster/card/Container.scss` — `.fm-card-state-line` (the named width-query context `state-line`), `.fm-card-lane-count`, `.fm-card-telltale`, `.fm-card-beacon` (yields below a 340px line — the one member with a width policy); the bands at `@container (max-width: 289px)` and `(max-width: 240px)`, the latter with the state line's visible wrap.
- `apps/agentos/view/fleet/roster/card/Container.mjs` — the state-line items and their texts; the line carries the vbox's inline `flex: 1 1 0%`, so a height on it needs `flex: 'none'` in the item config.
- `apps/agentos/util/Telltale.mjs` — `describeTelltale` owns `{text, title, ariaLabel, hidden}` as one unit (CARD-CONTRACT: the card never composes telltale prose).
- `apps/agentos/view/fleet/shared/StateDotComponent` — a 3px halo around the dot: the line may clip its block axis only.
- Precedent: the cockpit bar's declared collapse order ("three designed forms, no mid-word clipping ever", marks-with-titles below 760px).
- Witnesses: `test/playwright/visual/FleetCockpitVisual.spec.mjs` (the 314 vessel arm: state word only), `test/playwright/e2e/agentos/AgentCardSynthesisRenderNL.spec.mjs` (matrix goldens + the beacon geometry arm).

## The Fix

One fit policy and two designed forms; no measurement in the component.

1. **Fit — one row, content-aware, SCSS only.** The state line wraps inside a line that is one row high and clips its block axis (`overflow-y: clip`; the inline axis stays visible for the dot's halo). A member that does not fit takes a row the line does not show: it leaves WHOLE, from the end — never cut, never under the verbs — and the card's height stops depending on a resident's data. This replaces the sub-narrow band's visible wrap.
2. **Order is priority:** dot · state word · telltale · band · beacon words · badge (right-pinned). The exception outranks the ordinary facts, so it sits where it leaves last. The component's items reorder, so DOM, visual and reading order agree. Clipped members stay in the accessibility tree.
3. **Forms by the line's width** (the `state-line` container):
   - telltale: the words from a 508px line (8 + 80.3 + 260.8 + 86.3 + 72.2 — the widest active line holds them); below it a mark naming the deviating axes (`w` · `t` · `w·t`), worded by `describeTelltale` beside the text it already owns, title and aria unchanged. A retracted empty chip was prototyped and rejected: beside a state word it reads as an unchecked checkbox.
   - badge: `N lanes` where the widest active line holds it beside the mark (≈ 270–285px, arithmetic beside the rule); below it `N`, the full phrase on title and aria.
   - beacon words: keep their 340px threshold and yield to a telltale on the same line (`:has()`).
4. **Witnesses:** a geometry arm across the matrix over the closed vocabularies' long rows; the vessel arm guards every member; goldens re-rendered where pixels change.

**Prototype evidence (intake):** 1–3 as injected CSS over the live app, 15 card widths (191–720) × 7 rows (control · widest active · both axes · single axis · `benched / offline · never connected` · beacon-absent with both axes · bare): 0 members past the line, 0 under the verbs, the line 16px at every width, card heights data-independent (143 sub-narrow · 108 narrow · 104 regular). From a 428px card every member of every row is visible; at 410 only the `benched / offline · never connected` row's badge leaves.

`Prescription checked: resources/scss/src/apps/agentos/fleet/roster/card/Container.scss (.fm-card-state-line) — owns the concern; the thresholds-only fix first filed here could not meet its own first criterion (finding 2), so the fit policy joins the same rule and the forms' wording goes to util/Telltale.mjs, its contract owner.`

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.fm-card-state-line` | the card's own width modes (ADR 0029) | one clipped-wrap row; members leave whole from the end; order = priority | — | card SCSS comment + CARD-CONTRACT note | geometry arm across the matrix |
| `Telltale.describeTelltale` | CARD-CONTRACT (the resolver owns the chip's wording as one unit) | adds `mark` (`w` · `t` · `w·t`); `text` · `title` · `ariaLabel` · `hidden` unchanged | hidden → `mark: ''` | JSDoc + CARD-CONTRACT row | unit arms |
| `.fm-card-telltale` | the `state-line` container | words from a 508px line, the mark below | nominal renders nothing (unchanged) | card SCSS | geometry arm; goldens |
| `.fm-card-lane-count` | the `state-line` container | `N lanes` where the line holds it, `N` + title/aria below | null or 0 → no badge (unchanged) | card SCSS + JSDoc + CARD-CONTRACT row | unit arm for the title/aria pair; geometry arm |

**Decision Record impact:** aligned-with ADR 0029 (card-owned width modes via `@container`); none otherwise.

## Acceptance Criteria

- [ ] Geometry arm in the synthesis spec, widths 191 · 240 · 271 · 294 · 314 · 319 · 320 · 410 · 500 · 668 over the seven rows above: every VISIBLE member ends inside the line and none intersects the verbs; the state word is visible and whole at every width; the line is one row high; every card in a render has the height of a control-only render at that width — asserted, not snapshotted.
- [ ] Priority: where not everything fits, members leave from the end — the badge first, the telltale last; at the 410 column floor the widest active line with both telltale axes shows every member.
- [ ] Forms: the telltale's words render from a 508px line and its mark below; the badge's noun renders where the rule's arithmetic says and the number below; title and aria-label carry the full phrase in both forms; the state word is untouched at every width.
- [ ] Unit arms: `describeTelltale().mark` per axis combination; the card's badge and telltale vdom (both forms present, title/aria pairs). No measurement enters the component.
- [ ] The vessel arm in `FleetCockpitVisual.spec.mjs` guards every member, not only the state word; the goldens that change are re-rendered from a full visual run and read by eye in the PR; `CARD-CONTRACT.md`'s badge and telltale rows and a fit-policy note are updated.

## Out of Scope

- The narrow band's 44px verb pair — it is what leaves the line 95–143px; if a phone-width roster needs more facts on the line, that is the verbs' decision, not the line's.
- The band's vocabulary (short forms, the `benched / offline` + `benched` redundancy, ` · validation stale` inside the band text).
- The roster's measured row height (#128) — this ticket only stops the state line from feeding data into it.

## Avoided Traps

- **Thresholds alone** — the vocabularies vary too much; see The Problem. Forms are priced per width, FIT is content-aware.
- **A visible second row** — since #128 it re-pitches the whole roster whenever one resident is dense; the roster's pitch must not depend on a resident's data (the same reason the work row keeps its reserve).
- **Ellipsis on chips** — a mid-word cut is clipped content; a member leaves whole or renders a designed form.
- **Measuring in the component** — the card is layout-blind by contract.
- **An anonymous mark** — an empty chip reads as a checkbox and cannot say WHICH axis deviates; the initials can.

## Related

#112 / PR #119 (the beacon words' policy and the `state-line` container) · #128 / PR #159 (measured rows) · #10 (the cockpit UI/UX epic) · #13 (design conformance) · `apps/agentos/CARD-CONTRACT.md`.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-12T10:03Z and again at 2026-09-18T17:01Z; no equivalent found. A2A in-flight claim sweep: no claim on the roster card or its state line. Memory Core rationale sweep (`query_raw_memories`, two queries on the line's members and on clipped-wrap / mark regimes): nothing on this surface; one adjacent lesson adopted — `container-type: inline-size` needs a definite inline axis, which the line has (it is stretched by the identity column and already is the #112 container). Own-assignment sweep: #129 (PR #167, chrome) — no overlap.

Intake 2026-09-18 (self-authored, earlier session; drift probe non-empty on every declared path → full gate): premise re-measured and broadened, prescription replaced (above), body rewritten in one edit.

Origin Session ID: fcdd7d71-e7bd-46e8-bd01-5d46ea620205 · intake session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59

Retrieval Hint: "roster card state line badge telltale under the verbs fit policy clipped wrap mark regime"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


## Timeline

- 2026-09-12T10:04:59Z @neo-fable-clio added the `bug` label
- 2026-09-12T10:04:59Z @neo-fable-clio added the `agent-os` label
- 2026-09-12T10:04:59Z @neo-fable-clio added the `ai` label
- 2026-09-12T10:04:59Z @neo-fable-clio added the `design` label
- 2026-09-12T13:17:21Z @neo-fable-clio cross-referenced by #127
- 2026-09-12T13:17:37Z @neo-fable-clio cross-referenced by #128
- 2026-09-12T13:17:50Z @neo-fable-clio cross-referenced by #129
- 2026-09-14T01:11:30Z @neo-gpt cross-referenced by #10
- 2026-09-18T17:04:01Z @neo-fable-clio changed title from **Roster card: the badge and the telltale run under the verbs at the narrow bands** to **Roster card: the state line runs under the verbs — it needs a fit policy, not only at the narrow bands**
- 2026-09-18T17:04:03Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T17:31:00Z @neo-fable-clio cross-referenced by #168
- 2026-09-18T18:32:12Z @neo-fable-clio cross-referenced by PR #169
- 2026-09-18T18:34:38Z @neo-fable-clio cross-referenced by #11
- 2026-09-19T10:32:05Z @neo-fable-clio cross-referenced by #170
- 2026-09-19T11:00:05Z @neo-fable-clio cross-referenced by #171
- 2026-09-19T12:57:18Z @tobiu referenced in commit `272e5af` - "Merge pull request #169 from neomjs/agent/123-state-line-fit-policy

fix(roster): the card's state line is one row that fits — members leave whole, never under the verbs (#123)"
- 2026-09-19T12:57:18Z @tobiu closed this issue
- 2026-09-19T13:04:49Z @neo-fable-clio cross-referenced by PR #172

