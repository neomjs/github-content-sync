---
id: 44
title: The memories pane's browse and drill render through buffered grids
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-08-28T23:04:37Z'
updatedAt: '2026-08-29T11:20:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/44'
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
closedAt: '2026-08-29T11:20:40Z'
---
# The memories pane's browse and drill render through buffered grids

## Context

The second scored implementation of the #20 arc ("we need to actually improve our app", operator 2026-08-28). The memories pane's information design is merged substrate (`apps/agentos/design/institution-memories-pane.html`, #37 + the #38 drill-label follow-up), #24's primitive map names the destination (law 0: "mailbox surfaces, memories and catch-up → grid.Container"), and Institution #40/#41 just established the working pattern on the mailbox: headerless component-column grid, ONE data path (`applyBags` — display facts stamped into plain bags before they become records), pane-owned sequential drain instead of paging chrome, and a mounted Grid→pooled-row seam test from day one.

## The Problem

`view/fleet/memories/Container.mjs` (710 lines) hand-rolls BOTH of its list registers: the browse register (session summary cards) and the drill register (one session's turn rows) render via `renderRows`/`buildCard`-style vdom assembly, and both page through `more` / `older-turns` button chrome — the exact chrome class the operator retired on buffered surfaces (2026-08-28, applied to the mailbox in #40/#41). `Neo.grid.Container` + `grid.column.Component` sit unused while the pane re-implements bounded rendering by hand.

## The Architectural Reality

- The pane is a projection surface under a coherence contract that MUST survive the conversion: summary cards render only for the selected target (`renderedTarget` gate — a stale or late foreign-target page can never resurrect old cards), turn rows only for the open drill session (`renderedDrillSession` gate), and `page.offset > 0` envelopes append only onto an already-accepted page zero of the SAME key. These guards are substance, not chrome — they move into the projection path unchanged.
- Stores and models exist: `store.AgentSessionSummaries` + `store.AgentSessionTurns` over `model.SessionSummary` / `model.SessionTurn` (prompt/thought/response; `miniSummary` does NOT exist on the wire today — neo-agent-brain#210 is the producer-side exposure fix).
- The merged sketch's row anatomies: summary card = kind · title · provenance chip (derived) · age (T5 ViewerTime, ISO on `title`) over a 2-line-clamped summary body, cards grouped under viewer-calendar recency bands; drill turn row = AUTHORED provenance (visually distinct from DERIVED cards), the turn title line, prompt as bounded secondary context. The sketch's measured law: the per-turn `miniSummary` (tweet-size) IS the turn title once the wire carries it; until then the bounded `response` head remains the title line (forward-compatible model field + fallback — never a phantom).
- The #41 pattern transfers wholesale: derived display facts (here: the calendar band label on the first card of each viewer-local band — the recency eyebrow) are stamped into plain bags via the grid's `stampBandFacts` analog BEFORE records exist, because the store data path filter/add renders cells immediately (measured on #41: facts arriving after the set miss the first paint). Every projection creates fresh record identities — that is what re-seats pooled cells; no record mutation, no version choreography.
- Data acquisition stays intent-driven: the pane fires read intents; the owning FleetCockpit executes. The drain replaces the buttons — while an accepted coherent envelope reports more corpus (`summaryStore.count < total`, and the drill twin), the pane fires exactly ONE follow-up read intent per received envelope, armed per envelope arrival so re-renders never re-request (the #41 drain contract, guard keys included).
- Engine seam: `GridDragScroll` swallows real pointer clicks on in-cell buttons (defect-note broadcast 2026-08-28, recorded in `MailboxGridSeam.spec.mjs`) — the drill-open affordance on summary cards is delegated exactly like the mailbox thread toggle, and the seam test drives it via `dispatchEvent('click')` with the seam documented.

## The Fix

1. New `AgentOS.view.fleet.memories.SummaryGrid` + `TurnGrid` (both extending `Neo.grid.Container`, headerless single component column — the #41 shape) with flat pooled `SummaryRow` / `TurnRow` components rendering from fresh `rowData` bags (same-instance records never re-fire afterSet; #41's docblocks carry the contract).
2. ONE data path per grid: `applyBags` + band-fact stamping; the browse grid's delegated click opens the drill (record resolution via the engine's `.neo-grid-row` `data.recordId` contract); the drill keeps its back affordance in the pane head.
3. The pane keeps: coherence guards (target/session keys), intent events, honest states (no-selection / unavailable-with-reason / empty / rows), drill open/close mechanics. The hand-rolled `renderRows`/card+turn builders, the `more` / `older-turns` paging cluster and their SCSS retire; the drain replaces them.
4. New SCSS under the §04 contract (T1 role tokens only — zero font-size literals stays the greppable acceptance; T2 chip geometry; `--fm-ink-dim` floor for information-bearing text; the header-toolbar height-collapse rule from mailbox `Grid.scss` — never `display: none`, the header is the engine's flex-column measuring instrument).
5. Tests at both layers from day one: unit (row grammar both registers, coherence guards, drain witnesses with construction-listener fires, honest states) + a mounted component seam spec per the `MailboxGridSeam.spec.mjs` pattern (render, recycle, delegated drill-open through a genuine bubbling DOM click).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `view.fleet.memories.SummaryGrid` / `TurnGrid` | #24 law 0/1 + merged #37/#38 sketch | buffered grids render the designed card/turn anatomies | n/a (new) | class JSDoc cites sketch + #41 pattern | unit + mounted seam spec |
| browse/drill paging chrome | operator direction 2026-08-28 (no paging chrome on buffered surfaces) | retired; pane-owned sequential drain (one intent per accepted envelope, coherence-keyed) | git history | drain contract in pane JSDoc | drain witnesses (fire-once, guard-keyed, stop at corpus end) |
| coherence contract (target/session keys) | existing pane JSDoc | unchanged, enforced in the projection path | n/a | existing docblocks carry over | unit: foreign-key envelope never renders/appends |
| turn title line | merged sketch measured law + neo-agent-brain#210 | `miniSummary` when the wire carries it; bounded `response` head until then | response head (today's behavior) | model field JSDoc names #210 | unit: title source switch |
| read-only mirror MUST-NOT | existing adapter contract | unchanged — no write path from this surface | n/a | existing JSDoc | unit: no mutation affordances |

## Decision Record impact

None — applies #24's operator-stated laws, the merged #37/#38 sketches, and the #41-established pattern.

## Acceptance Criteria

- [ ] `SummaryGrid` + `TurnGrid` extend `Neo.grid.Container` (headerless component columns) rendering the sketch's card/turn anatomies from the existing stores; browse cards group under viewer-calendar band facts stamped at bag time.
- [ ] ONE data path per grid (`applyBags`): every projection stamps facts into plain bags and sets store data wholesale; no record mutation anywhere in the view layer.
- [ ] Coherence guards survive: a foreign-target summary envelope and a foreign-session drill envelope neither render nor append; `offset > 0` extends only an accepted page zero of the same key.
- [ ] No paging chrome anywhere; the drain fires exactly one follow-up read intent per accepted envelope while more corpus exists, and stops at the honest end — red-capable witnesses for both registers.
- [ ] T5: zero raw-ISO human strings (ViewerTime + ISO `title`); T2 exception-only chips; turn rows visually AUTHORED vs the cards' DERIVED provenance.
- [ ] Turn title = `miniSummary` when present, bounded response head otherwise (forward-compatible field, JSDoc names neo-agent-brain#210).
- [ ] The hand-rolled row/card builders + `more`/`older-turns` cluster and their SCSS retire; zero dangling references.
- [ ] Zero font-size literals in the new SCSS (T1 role tokens, both themes).
- [ ] Mounted component seam spec (MailboxGridSeam pattern): render, recycle, delegated drill-open via genuine bubbling DOM click — deterministically green 3 consecutive runs.
- [ ] Full owning unit tree green; no unrelated visual baseline refreshed (AC-7 restamp per commit).

## Out of Scope

- The search and bird-view registers of the merged sketch (gated on fleet-source exposure of `query_summaries`/`explore_memory_history` — their own leaves once the Brain side exposes the ops).
- neo-agent-brain#210 itself (producer-side `miniSummary` exposure — this ticket only lands the consumer fallback).
- The catch-up pane (its own #24 law-0 leaf).
- Any mailbox file (PR #41 is in review; zero overlap by construction).

## Related

Parent: #24 · Pattern: #40 / PR #41 (mailbox grid, one data path, drain, seam spec) · Spec: merged `institution-memories-pane.html` (#37, #38) · Producer dep (soft): neo-agent-brain#210 · Engine seams: neomjs/neo#17835 (scroll-edge), GridDragScroll defect-note (2026-08-28 broadcast).

Live latest-open sweep: checked latest 20 open issues at 2026-08-28T23:04Z — no equivalent (closest: #20 design arc [merged sketches], #24 parent map). A2A in-flight claim sweep: last-hour claims cover Brain #184/#200, Skills #13, DockLayouts #17838/#39 — zero overlap with the memories surface.

Origin Session ID: 41859592-b7ee-4bce-bee3-f25644d9003b

Retrieval Hint: `query_raw_memories("memories pane grid one data path applyBags band facts drain")`

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-28T23:04:39Z @neo-fable-clio added the `enhancement` label
- 2026-08-28T23:04:39Z @neo-fable-clio added the `ai` label
- 2026-08-28T23:04:39Z @neo-fable-clio added the `design` label
- 2026-08-28T23:04:44Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T23:20:53Z @neo-fable-clio cross-referenced by PR #45
- 2026-08-28T23:34:26Z @tobiu referenced in commit `82f9299` - "fix(agentos): height-norm the memories cells for the fixed row lattice (#44)

Browser design pass on the mounted registers (the operator's design-read
law: never ship a surface whose defects went unseen) caught three real
classes:

1. The engine grid positions rows on a FIXED rowHeight lattice (32px
   default) — variable card heights overlapped cell content onto itself
   (measured: ~180px cards on the 32px lattice). Cells are now
   height-NORMED: band slot on EVERY card (empty off boundaries),
   attribution rides the one meta line (ahead of the counters — who else
   authored outranks a quality figure when the clamp cuts), title 1-line
   ellipsis, body 2-line clamp; SummaryGrid rowHeight 132 / TurnGrid 104
   carry the same totals as the SCSS heights (each side names the other).
2. The engine loads CSS per instantiated CLASS NAME — a memories/Grid.css
   maps to no class (SummaryGrid/TurnGrid) and never loads; the rules
   live in the pane's always-present Container.scss now, with the reason
   recorded where a future reader would recreate the trap.
3. Titled turns (miniSummary) overflowed the normed cell and clipped the
   prompt mid-glyph — beside a title the response clamps to one context
   line (sibling selector); drill head gains its breathing room.

Specs pinned to the normed contract (band slot always present, meta-line
attribution, seam band assert by label); 716 unit + component 3/3 + 6
visual green."
- 2026-08-28T23:37:20Z @tobiu referenced in commit `3464997` - "fix(agentos): height-norm the mailbox rows for the fixed row lattice (#40)

Mounted browser design pass (run for the memories twin on #44, applied
here the same hour): the engine grid positions rows on a FIXED rowHeight
lattice — the designed mailbox rows measured 39-84px on the 32px default,
overlapping cell content onto itself. The row is normed to the tallest
designed case (sender + 2-line subject clamp + exception strip): Grid
rowHeight 84 and the .fm-mail-row height carry the same total, each side
naming the other; quieter rows keep the rhythm with whitespace, overflow
clamps, never grows. 716 unit + component 3/3 + 6 visual green."
- 2026-08-28T23:39:53Z @tobiu referenced in commit `88ee615` - "fix(agentos): law-1 suffixes for the memories row cells (#44)

viewTopologyConformance (#17559 law 1) rejects a component.Base subclass
whose suffix is not 'Component': SummaryRow/TurnRow rename to
SummaryRowComponent/TurnRowComponent (the mailbox RowComponent precedent).
Caught by CI, then reproduced locally — the earlier 'green' read piped
the Playwright summary through tail -1, which drops the '1 failed' line
above the passed line; both summary lines are the only honest read.
Full battery 717 passed / 0 failed, component 3/3, visual 6/6."
- 2026-08-28T23:42:31Z @neo-fable-clio cross-referenced by #24
- 2026-08-29T00:51:15Z @tobiu referenced in commit `716ac8e` - "docs(agentos): repair three Anchor & Echo references (#44)

Emmy's RA-1 on PR #45: the night's own refactors left three durable
links pointing at retired identities — the rowHeight contracts in
SummaryGrid/TurnGrid named the deleted fleet/memories/Grid.scss (its
rules live in Container.scss since the CSS-per-className finding), and
TurnRowComponent still linked SummaryRow from before the law-1 rename.
All three now name the delivered identities; whole-repo grep clean.
No runtime or test change; 717/0 unit."
- 2026-08-29T09:57:14Z @tobiu referenced in commit `383259e` - "feat(agentos): memories browse + drill render through buffered grids (#44)

The second scored #20-arc surface, on the #40/#41 pattern: the 710-line
hand-rolled memories pane's two list registers move onto headerless
component-column grids.

- RowsGrid (base): the ONE data path — applyBags stamps derived display
  facts into PLAIN bags before they become records (the store data path
  renders during add; late facts miss the first paint) and every
  projection creates fresh record identities, which is what re-seats
  pooled cells. extractBags reads the corpus back for window extension.
- SummaryGrid: viewer-calendar band facts (first card of each band
  carries the eyebrow — stamped once, at bag time) + the delegated
  drill-open click resolving records via the engine .neo-grid-row
  data.recordId contract, re-fired as the cardOpen pane intent.
- TurnGrid + SummaryRow/TurnRow: flat pooled cells from fresh rowData
  bags; the turn title follows the sketch's measured law — miniSummary
  once the wire carries it (Brain #210; forward-compatible model field),
  bounded response head until then.
- The pane keeps the coherence contract unchanged (target/session keys,
  offset>0 extends only an accepted page zero of the same key) and
  DRAINS both registers: one follow-up intent per newly-arrived accepted
  envelope while total says more exists, floored by rendered depth so a
  repeated answer can never loop. The more/older-turns paging chrome and
  the hand-rolled card/turn builders retire; Refresh stays (an explicit
  re-read intent is not paging).
- SCSS: grid header-collapse (never display:none — the header toolbar is
  the engine's flex-column measuring instrument), cell structure, band
  eyebrow, native-button affordance; zero font-size literals.
- Tests: 22 unit (coherence + drain witnesses + row grammar both
  registers + hostile text-only) and the mounted MemoriesGridSeam
  component spec from day one — render, recycle, real delegated
  drill-open (dispatchEvent per the GridDragScroll engine seam), back
  round-trip; 3/3 consecutive green."
- 2026-08-29T09:57:15Z @tobiu referenced in commit `24b9d0d` - "fix(agentos): height-norm the memories cells for the fixed row lattice (#44)

Browser design pass on the mounted registers (the operator's design-read
law: never ship a surface whose defects went unseen) caught three real
classes:

1. The engine grid positions rows on a FIXED rowHeight lattice (32px
   default) — variable card heights overlapped cell content onto itself
   (measured: ~180px cards on the 32px lattice). Cells are now
   height-NORMED: band slot on EVERY card (empty off boundaries),
   attribution rides the one meta line (ahead of the counters — who else
   authored outranks a quality figure when the clamp cuts), title 1-line
   ellipsis, body 2-line clamp; SummaryGrid rowHeight 132 / TurnGrid 104
   carry the same totals as the SCSS heights (each side names the other).
2. The engine loads CSS per instantiated CLASS NAME — a memories/Grid.css
   maps to no class (SummaryGrid/TurnGrid) and never loads; the rules
   live in the pane's always-present Container.scss now, with the reason
   recorded where a future reader would recreate the trap.
3. Titled turns (miniSummary) overflowed the normed cell and clipped the
   prompt mid-glyph — beside a title the response clamps to one context
   line (sibling selector); drill head gains its breathing room.

Specs pinned to the normed contract (band slot always present, meta-line
attribution, seam band assert by label); 716 unit + component 3/3 + 6
visual green."
- 2026-08-29T09:57:15Z @tobiu referenced in commit `6a6f52d` - "fix(agentos): law-1 suffixes for the memories row cells (#44)

viewTopologyConformance (#17559 law 1) rejects a component.Base subclass
whose suffix is not 'Component': SummaryRow/TurnRow rename to
SummaryRowComponent/TurnRowComponent (the mailbox RowComponent precedent).
Caught by CI, then reproduced locally — the earlier 'green' read piped
the Playwright summary through tail -1, which drops the '1 failed' line
above the passed line; both summary lines are the only honest read.
Full battery 717 passed / 0 failed, component 3/3, visual 6/6."
- 2026-08-29T09:57:15Z @tobiu referenced in commit `f1fa006` - "docs(agentos): repair three Anchor & Echo references (#44)

Emmy's RA-1 on PR #45: the night's own refactors left three durable
links pointing at retired identities — the rowHeight contracts in
SummaryGrid/TurnGrid named the deleted fleet/memories/Grid.scss (its
rules live in Container.scss since the CSS-per-className finding), and
TurnRowComponent still linked SummaryRow from before the law-1 rename.
All three now name the delivered identities; whole-repo grep clean.
No runtime or test change; 717/0 unit."
- 2026-08-29T11:20:40Z @tobiu referenced in commit `76a7082` - "Merge pull request #45 from neomjs/agent/44-memories-grid

feat(agentos): memories browse + drill render through buffered grids (#44)"
- 2026-08-29T11:20:40Z @tobiu closed this issue
- 2026-08-29T21:51:22Z @neo-gpt-emmy cross-referenced by PR #54
- 2026-09-12T16:48:04Z @neo-fable-clio cross-referenced by #131
- 2026-09-12T17:08:04Z @neo-fable-clio cross-referenced by #18626
- 2026-09-12T17:09:33Z @neo-fable-clio cross-referenced by PR #132

