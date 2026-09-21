---
id: 40
title: The mailbox pane becomes AgentOS.view.fleet.mailbox.Grid
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - refactoring
assignees:
  - neo-fable-clio
createdAt: '2026-08-28T21:31:47Z'
updatedAt: '2026-08-29T09:50:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/40'
author: neo-fable-clio
commentsCount: 0
parentIssue: 24
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-29T09:50:02Z'
---
# The mailbox pane becomes AgentOS.view.fleet.mailbox.Grid

## Context

Operator direction after the #20 sketch arc merged (2026-08-28): "a lot of specs, but we need to actually improve our app 🙂" — the spec-first gates are passed; this is the first scored implementation. The mailbox pane's information design is merged substrate (`apps/agentos/design/institution-mailbox-pane.html`, #36 + the #38 follow-up), and #24's primitive map names the destination: mailbox surfaces → `grid.Container`, law-1 name `AgentOS.view.fleet.mailbox.Grid`.

## The Problem

The shipped mailbox surface (`view/fleet/mailbox/Container.mjs`, 691 lines + `OperatorContainer.mjs`) hand-rolls its rows (subject span + one joined meta string with raw ISO — a T5 violation), hand-rolls offset paging (`newer/older` — retired by operator direction), and extends `Container` against #24 law 0 while `Neo.grid.Container` (buffered rendering, store binding, selection/keyboard) sits unused.

## The Architectural Reality

- `Neo.grid.Container` + `grid.column.Component` (pooled cell components, `cellDepth` scroll-sizing) provide the buffered row machinery; the sketch's row anatomy is ONE coherent row object — the natural shape is a single headerless Component column rendering the designed row (dot · monogram · content · age), with the implementer free to split columns if measurement argues for it.
- The merged sketch is the scored spec: subject = the one body-tier line with read-state weight (unread dot + 600), T2 exception-only chips (priority high, task envelope, broadcast class, ticket tail), T5 ViewerTime age with ISO `title` (`util/ViewerTime.mjs` exists), thread collapse (head + `+N earlier`, rail-indented members) as row-level native buttons, retracted = line-through on readable ink, `--ink-dim` contrast floor, Institution identity.
- No paging chrome: the buffered surface scrolls, and the pane drains the remote corpus itself — one sequential window request per received snapshot while the producer reports `hasMore`, silence at the honest end — until neomjs/neo#17835 generalizes the scroll-edge seam engine-side.
- Adapter guarantees bound the states — no phantom classes (sentAt server-generated, from authenticated, per MailboxService): the mirror adapter always answers with a capability/admission envelope, so the pane's user-facing family is `unobserved` (no snapshot observed yet) · `denied` (admission refused, named) · `degraded` (source degraded / fail-closed, adapter reason) · `empty` (granted, zero rows) · `rows`. There is no separate loading state (between polls the last snapshot + the freshness chip ARE the truth) and no pane-side retry (the next poll is the retry — poll cadence is the owning controller's contract).
- The store/model layer stays: `AgentOS.model.MailboxMessage` + `store.AgentMailbox` (read-only mirror MUST-NOT untouched — no mark-read from this surface).
- Per #24: the new surface is born conformant in `view/fleet/mailbox/`; the replaced hand-rolled view retires in this PR ("a new view that replaces an old file retires that file in its own PR"). `ComposeForm`/`RecipientChip*` (already law-1-suffixed) stay.

## The Fix

1. New `AgentOS.view.fleet.mailbox.Grid` extending `Neo.grid.Container`: headerless, single Component column rendering the sketch's row anatomy from `MailboxMessage` records; grid-owned selection/focus/keyboard; thread-collapse as the row's native button toggling the view-owned `threadCollapsed` display state.
2. New SCSS under the Institution identity (dim contrast floor, T1 role tokens, T2 chip geometry — zero font-size literals is the pass's greppable acceptance).
3. The composing surfaces (south pane / operator container seams) mount the Grid; the hand-rolled row rendering + offset-paging controls retire with their SCSS.
4. Controller keeps snapshot/scope mechanics; the offset paging handlers retire in favor of the pane-owned drain: while a received snapshot reports `page.hasMore`, the pane fires exactly ONE follow-up window request (`pageRequest`, relayed unchanged by the operator container), and follow-up windows extend the held corpus — row 51+ stays reachable with zero chrome, and `hasMore: false` is the only stop.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `view.fleet.mailbox.Grid` | #24 law 0/1 + the merged sketch | buffered grid renders the designed rows | n/a (new) | class JSDoc cites the sketch | unit: row anatomy, chips exception-only, thread toggle, T5 age |
| hand-rolled mailbox view | #24 retirement rule | retired in this PR | git history | removal noted in PR | zero references remain |
| read-only mirror MUST-NOT | existing model/adapter contract | unchanged — no write path from the surface | n/a | existing JSDoc | unit: no mark-read call sites |

## Decision Record impact

None — applies #24's operator-stated laws and the merged #20 sketches.

## Acceptance Criteria

- [ ] `AgentOS.view.fleet.mailbox.Grid` extends `Neo.grid.Container`, renders the sketch's row anatomy from the existing store, and owns selection/focus/keyboard.
- [ ] T5: zero `toISOString()`-derived human strings; ages via `ViewerTime` with ISO on `title`.
- [ ] T2 exception-only chips: a direct/normal/plain/read message renders two quiet lines and zero chips.
- [ ] Thread collapse works as row-level native buttons (aria-expanded), rail-indented members.
- [ ] No paging chrome anywhere; the five adapter-honest user-facing states render (unobserved/denied/degraded/empty/rows), and a `hasMore: true` snapshot drains beyond the initial window (row 51+ reachable, `hasMore: false` the only stop).
- [ ] The replaced hand-rolled mailbox view and its paging SCSS are retired in this PR; zero dangling references.
- [ ] Zero font-size literals in the new mailbox SCSS (all five T1 roles via tokens); both themes via tokens.
- [ ] Full owning unit tree green; no unrelated visual baseline refreshed (AC-7 restamp ritual per commit).

## Out of Scope

- The S5 mirror-scope re-entry (subject switching — #20's design note, gated on Fleet grants).
- Compose placement decision (inline vs south tab — operator decision slot, unchanged).
- Memories/catch-up implementations (their own leaves).
- Engine #17835 (the generalized scroll-edge seam).

## Related

Parent: #24 (per-view primitive migration leaf) · Spec: the merged `institution-mailbox-pane.html` (#36, #38) · Design arc: #20 · Engine seam: neomjs/neo#17835.

Live latest-open sweep: checked latest 20 open issues at 2026-08-28T21:32Z — no equivalent. A2A in-flight claim sweep: mailbox drained to zero minutes prior, no overlapping lane-claim.

Origin Session ID: 4f07d934-f43f-4406-98a0-9562e122e471

Retrieval Hint: `query_raw_memories("mailbox Grid component column implementation scored sketch")`

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.



## Timeline

- 2026-08-28T21:31:48Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T21:31:49Z @neo-fable-clio added the `enhancement` label
- 2026-08-28T21:31:49Z @neo-fable-clio added the `ai` label
- 2026-08-28T21:31:49Z @neo-fable-clio added the `architecture` label
- 2026-08-28T21:31:49Z @neo-fable-clio added the `refactoring` label
- 2026-08-28T21:57:48Z @tobiu referenced in commit `c0b27de` - "feat(agentos): the mailbox pane renders through the buffered grid — paging chrome retires (#40)"
- 2026-08-28T21:58:32Z @neo-fable-clio cross-referenced by PR #41
- 2026-08-28T22:58:05Z @tobiu referenced in commit `6d96cc6` - "feat(agentos): mailbox rows ride ONE data path — drain, bag-stamped facts, seam witness (#40)

Emmy's three RAs on PR #41, closed in one architectural move:

RA-1: the pane drains the remote corpus itself — exactly one follow-up
window request per received snapshot while the producer reports hasMore
(armed per afterSetSnapshot, so freshness re-renders never re-request);
follow-up windows extend the held corpus. Red-capable witness: 50 rows +
hasMore true fires offset 50, appends to 60, hasMore false stops.

RA-3: MailboxGridSeam.spec.mjs mounts the pane through the REAL pipeline
(grid.Container -> grid.column.Component -> pooled RowComponent) in a
browser, proves render, recycle-with-threadFacts, and the delegated
thread toggle through a genuine bubbling DOM click — expand and collapse.

The enabling architecture (measured, not theorized): every mutation —
wholesale projection, window extension, thread toggle — flows through
Grid#applyBags: thread facts stamp into PLAIN bags before they become
records (the data path filter-during-add renders immediately; facts
arriving later missed the first paint), and every run creates new record
identities, which is what re-seats pooled cells. The former per-record
stamp choreography (record.set version bumps + store.filter) fired two
overlapping vdom transactions and double-mounted the head cell's content
into its own row. threadMap/buildThreadMap/threadFactsFor/onStoreMutation
and store#applySnapshotRows retire; the collapse filter reads stamped
record truth only.

RA-2 lands as the #40 author-correction (adapter-honest state family +
drain contract in body/ledger/AC); the PR body Deltas section carries
the disposition."
- 2026-08-28T23:04:38Z @neo-fable-clio cross-referenced by #44
- 2026-08-28T23:37:19Z @tobiu referenced in commit `3464997` - "fix(agentos): height-norm the mailbox rows for the fixed row lattice (#40)

Mounted browser design pass (run for the memories twin on #44, applied
here the same hour): the engine grid positions rows on a FIXED rowHeight
lattice — the designed mailbox rows measured 39-84px on the 32px default,
overlapping cell content onto itself. The row is normed to the tallest
designed case (sender + 2-line subject clamp + exception strip): Grid
rowHeight 84 and the .fm-mail-row height carry the same total, each side
naming the other; quieter rows keep the rhythm with whitespace, overflow
clamps, never grows. 716 unit + component 3/3 + 6 visual green."
- 2026-08-28T23:42:31Z @neo-fable-clio cross-referenced by #24
- 2026-08-29T00:47:39Z @neo-gpt-emmy cross-referenced by PR #45
- 2026-08-29T09:50:03Z @tobiu referenced in commit `c3531c0` - "Merge pull request #41 from neomjs/agent/40-mailbox-grid

feat(agentos): the mailbox renders through the buffered grid (#40)"
- 2026-08-29T09:50:03Z @tobiu closed this issue
- 2026-08-29T09:57:15Z @tobiu referenced in commit `383259e` - "feat(agentos): memories browse + drill render through buffered grids (#44)

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

