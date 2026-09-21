---
id: 168
title: 'Roster card: a long resident name is cut mid-glyph, never ellipsized'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-18T17:30:59Z'
updatedAt: '2026-09-19T13:30:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/168'
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
closedAt: '2026-09-19T13:30:26Z'
---
# Roster card: a long resident name is cut mid-glyph, never ellipsized

## Context

On a narrow roster card a long resident name is cut in the middle of a glyph, with no ellipsis, and the members after it on the name line — the name-provenance chip and the engine tag — are cut or invisible. The committed matrix goldens show it (`agentcard-synthesis-*-{294,319,320,328}`: "Abelard Fontaine-Marchb|", "Alexander Consta|"). The card's stylesheet says the opposite should happen: `.fm-card-name` declares `flex: 0 1 auto; min-width: 0; text-overflow: ellipsis` — "lets the name ellipsize instead of overflowing the card".

Measured 2026-09-18 on Institution dev@c100571 + the #129/#123 stack, under the Neural Link with the card-width matrix (`AgentCardSynthesisRenderNL.spec.mjs`'s pinning path), name `Alexander Constantine Maximilianus` (245px), engine tag 211px:

| card | name line | name box | name scroll / client | chip | engine tag |
|---|---|---|---|---|---|
| 191 | 129px | 245px | 245 / 245 | 124px past the line: invisible | hidden by its band |
| 294 | 118px | 245px | 245 / 245 | invisible | hidden by its band |
| 328 | 168px | 245px | 245 / 245 | invisible | invisible |
| 410 | 250px | 245px | 245 / 245 | starts 3px past the line: invisible | invisible |
| 720 | 560px | 245px | 245 / 245 | visible | visible |

Beside a short name (`Ada`) the chip is fine, and the 211px tag is cut mid-word at 108px (328 card) and 190px (410 card).

The name is 245px wide at EVERY card width and never overflows ITSELF (scroll = client), so its ellipsis can never engage; `.fm-card-name-line`, `.fm-card-identity` and `.fm-card-head` clip it instead (`overflow-x: hidden` on all three). Computed `flex` on the name: `0 0 auto`, from the inline style `flex: 0 0 auto;`.

Promotion of my defect-note (`MESSAGE:b8a6e256`, 2026-09-18) by triage decision: it is "clipped content", the form the card contract forbids, on the cockpit's atom.

## The Problem

The name item's config says `flex: 'none'`, and a Neo item's `flex` config is an INLINE style: it beats the stylesheet's `flex: 0 1 auto`, so the shrink the ellipsis depends on never happens. The name line also has no fit policy: every member is rigid, so whatever does not fit is cut by an ancestor at an arbitrary pixel — and a rigid long name pushes the chip and the tag out even where they would fit beside an ellipsized one. The state line had the same class of defect (#123); the two rows of the identity column should share one policy.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/card/Container.mjs` — the `name-line` container (hbox) and its items `card-name` · `name-provenance` · `card-engine`, all `flex: 'none'`; the line itself carries the identity vbox's inline `flex: 1 1 0%`.
- `resources/scss/src/apps/agentos/fleet/roster/card/Container.scss` — `.fm-card-name-line` (`gap`, `min-width: 0`), `.fm-card-name` (the ellipsis rule that cannot fire), `.fm-name-provenance`, `.fm-card-engine` (hidden at the narrow band).
- #123 (the state line's fit): a wrapping line that shows its first row only, so a member the row cannot hold leaves whole from the end; item order = priority.
- Witnesses: `test/playwright/e2e/agentos/AgentCardSynthesisRenderNL.spec.mjs` (matrix goldens; the fit arm from #123), `test/playwright/unit/apps/agentos/view/fleet/roster/card/container.spec.mjs`.

## The Fix

The name line takes the state line's policy, with one difference: its first member may shrink.

1. `card-name`: `flex: '0 1 auto'` in the item config, so the stylesheet's ellipsis rule is what renders. The name is the row's subject: alone on the row it shrinks to the line and ellipsizes.
2. The name line wraps (`wrap: 'wrap'`, `flex: 'none'`) and shows its first row only (`height: 1lh` on the display font, `overflow-y: clip`): the chip and the engine tag stay where they fit beside the name and leave WHOLE where they do not — never cut. Order = priority: name · chip · engine tag (already the item order).
3. The full name stays reachable: the name carries its text as `title` when it can be ellipsized.

**Prototype evidence (before filing):** 1–2 as injected CSS over the live app, widths 191 · 294 · 328 · 410 · 720: the long name ellipsizes to the line at 191 · 294 · 328 (scroll 245 > client 129 · 118 · 168) and renders whole from 410; a short name keeps its chip at every width; the 211px tag leaves whole until 720; the name line is 18.2px at every width and card heights stay data-independent (143 · 108 · 104).

`Prescription checked: apps/agentos/view/fleet/roster/card/Container.mjs (the name item's flex config) + resources/scss/src/apps/agentos/fleet/roster/card/Container.scss (.fm-card-name-line) — own the concern; shrinking every member instead (nowrap) was rejected by arithmetic: the rigid chip and tag would make the NAME pay for the row, the inverse of the priority.`

Contract Ledger: not applicable — item configs and the stylesheet only; no consumed surface changes. **Decision Record impact:** aligned-with ADR 0029 (card-owned width modes).

## Acceptance Criteria

- [ ] Across the card-width matrix (191 · 294 · 328 · 410 · 720) with a 245px name: the name ends inside the name line; where it does not fit it is ellipsized (scroll width > client width, `text-overflow: ellipsis`), never cut by an ancestor — asserted, not snapshotted.
- [ ] Every VISIBLE member of the name line ends inside the line; the chip and the engine tag are whole or absent, never cut; with a short name the chip is visible at every width.
- [ ] The name line is one row high at every width, and a card's height does not change with its resident's name length.
- [ ] An ellipsized name is reachable in full (title); unit arm on the item configs (`flex`, the line's `wrap`) and the title.
- [ ] Matrix goldens re-rendered where pixels change and read by eye in the PR; `CARD-CONTRACT.md`'s display-name row notes the policy.

## Out of Scope

- The state line (#123) and the verbs' band rules.
- The engine tag's own vocabulary or a short form for it.

## Avoided Traps

- **Shrinking every member (nowrap)** — flex cannot express "the tag yields completely before the name loses a letter"; ratios make the name pay first.
- **An ellipsis on the chip or the tag** — a member leaves whole or renders a designed form (#123's rule).
- **Removing the ancestors' clip** — it would paint the name over the verbs; the clip is the symptom's messenger, not its cause.

## Related

#123 (the state line's fit policy — same mechanism, local commit c964bba, PR follows #167) · #10 (the cockpit UI/UX epic) · `apps/agentos/CARD-CONTRACT.md`.

Live latest-open sweep: checked the open Institution issues (16, created-descending) at 2026-09-18T17:29Z; no equivalent found. A2A in-flight claim sweep (last 30 messages, all read-states): no claim on the roster card or its name line besides my own defect-note. Memory Core rationale sweep (`query_raw_memories`, "resident name cut off mid-word without ellipsis on narrow agent card, provenance chip and engine tag invisible"): nothing on this symptom. Own-assignment sweep: #129, #123, #10 — bodies read; #123 covers the state line only.

Origin Session ID: 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59

Retrieval Hint: "roster card name line ellipsis inline flex none defeats stylesheet shrink clipped-wrap row"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 1ef6c04a-10f6-4977-ad7d-0e2c7f343c59


## Timeline

- 2026-09-18T17:30:59Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-18T17:31:01Z @neo-fable-clio added the `bug` label
- 2026-09-18T17:31:01Z @neo-fable-clio added the `agent-os` label
- 2026-09-18T17:31:01Z @neo-fable-clio added the `ai` label
- 2026-09-18T17:31:01Z @neo-fable-clio added the `design` label
- 2026-09-18T18:32:12Z @neo-fable-clio cross-referenced by PR #169
- 2026-09-18T18:34:38Z @neo-fable-clio cross-referenced by #11
- 2026-09-19T10:32:05Z @neo-fable-clio cross-referenced by #170
- 2026-09-19T11:00:05Z @neo-fable-clio cross-referenced by #171
- 2026-09-19T13:04:49Z @neo-fable-clio cross-referenced by PR #172
- 2026-09-19T13:30:26Z @tobiu referenced in commit `17ca2ba` - "fix(roster): a long resident name ellipsizes inside its line — the name line is one row that fits (#168)

The name item's flex config was 'none': an inline style that beat the stylesheet's shrink rule, so the name stayed 245px at every card width, its ellipsis could never fire, and ancestors cut it mid-glyph while the provenance chip and the engine tag were pushed out. The name is the row's one shrinkable member now, and the name line takes the state line's policy: it wraps and shows its first row only, so the chip and the tag leave whole where they do not fit. The title keeps an ellipsized name reachable."
- 2026-09-19T13:30:27Z @tobiu referenced in commit `0b41e48` - "Merge pull request #172 from neomjs/agent/168-name-line-fit

fix(roster): a long resident name ellipsizes inside its line — the name line is one row that fits (#168)"
- 2026-09-19T13:30:27Z @tobiu closed this issue

