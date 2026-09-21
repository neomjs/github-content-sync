---
id: 20
title: 'FM pane information design: reviewed per-surface design contracts against the #24 target primitives'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-08-16T22:31:20Z'
updatedAt: '2026-08-29T22:08:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/20'
author: neo-fable-clio
commentsCount: 6
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
closedAt: '2026-08-29T22:08:10Z'
---
# FM pane information design: reviewed per-surface design contracts against the #24 target primitives

# FM pane information design: the reviewed per-surface design contracts (design authority under #24)

## Context

Operator critique 2026-08-16, verbatim intent: "Mailbox view design? memories and summaries design and ux?" — the panes render CORRECT data as undesigned lists. The §04 arc (neomjs/neo#17263 spec + neomjs/neo#17264/#17265 chrome passes) covers the cockpit's shared visual language; this ticket owns the CONTENT-level information design of the three list-bearing surfaces.

**Re-scoped 2026-08-29** after the Euclid intake (needs-relinking) + Emmy triage (needs-re-triage) below: the live parent is #24 (the component-library conformance epic), whose Law 0 replaces the current hand-mapped row trees with real grid/list primitives. Styling those trees would polish structures #24 retires — so this ticket's deliverable is the **reviewed per-surface design contract against the TARGET primitives**, and the implementation belongs to #24's per-view migration leaves, each scored against its contract here.

## The deliverable — three design contracts, sketch-first

Per surface, against neomjs/neo#17263's ladders and #24's primitive map (mailbox, memories → `Neo.grid.Container`; catch-up refined to `Neo.list.Base` + `useHeaders` per its contract):

1. **Mailbox** (`grid.Container` target): message-row anatomy (sender identity mark, subject weight, relative age, read/unread state, priority), thread grouping, compose affordance placement — incl. the compose recipient-picker chrome (operator finding 2026-08-21: "chips look decent, list content needs design love"). Sketch landed: `apps/agentos/design/institution-mailbox-pane.html` — to be re-scored against the grid target (row anatomy maps to grid columns/cell components, not nested containers).
2. **Memories** (`grid.Container` target): entry cards with kind/recency scan pattern, session grouping + drill-in, provenance pills, the retrieval-hint affordance. Sketch landed: `apps/agentos/design/institution-memories-pane.html` — same grid re-score.
3. **Catch-up** (`Neo.list.Base` + `useHeaders` target — the contract's recorded refinement of the epic's grid direction: two fixed heterogeneous sections, nothing to buffer): partition hierarchy with visual weight, source-slot freshness as designed metadata rather than appended text. Contract landed: `apps/agentos/design/institution-catchup-pane.html`.

What "absorbed" means for the contracts: `#17219`/`#17491`/`#17340` already landed parts of the anatomy (subject/meta/unread rows, memory cards with provenance, partition/source hierarchy). The contracts codify that landed direction where it holds and mark the deltas the grid migrations must carry — they never re-litigate landed semantics.

## Acceptance Criteria

- [ ] Each of the three surfaces has a design contract (§04-consistent, target-primitive-aware) in `apps/agentos/design/`, cross-family-reviewed BEFORE its #24 migration leaf starts.
- [ ] The two pre-rescope sketches (mailbox, memories) are RE-SCORED against their `grid.Container` targets — row anatomy mapped to grid columns/cell components — the named remaining deliverable after the catch-up contract landed.
- [ ] Each contract names its implementation coordinate (the #24 leaf that consumes it) and states what the landed anatomy already covers vs. what the migration must add.
- [ ] The compose recipient-picker design is owned HERE (part of the mailbox contract); its implementation rides the mailbox leaf.
- [ ] Shell-seam rules restated per contract: pane roots frame-free, zero CSS-in-JS, both themes via tokens.

## Out of Scope

Implementation (the #24 per-view migration leaves own it, scored against these contracts) · the shared chrome ladders (neomjs/neo#17263-#17265) · navigation/deep-link routing and the AgentDetail status/refresh mechanics (accreted in comments below; they cross different owners — they get their own coordinates under #24/#23 when picked up, and are explicitly NOT folded into these contracts) · new data capabilities.

## Design note — the per-agent mailbox scope (2026-08-28, with #30)

#30 retired the agent-detail Mailbox tab: the south pane is the ONE mailbox surface. The retired per-agent MIRROR seam (`fleetMailboxMirror`, admission-gated, fail-closed — policy-held at `awaiting-s5`) re-enters through the mailbox CONTRACT when the S5 grants/admission layer lands: **one pane, two scopes** — a selection-scoped subject mode (operator inbox ↔ selected resident's mirror), not a second surface. The possession semantics (generation latch, subject re-check, fail-closed reads) are preserved verbatim in git history at the #30 removal commit and bind the contract.

## Related

Parent: #24 (component-library conformance — Law 0 primitive map; this ticket is its named design authority per the Epic Review ask). Grandparent: #10. The §04 bar: neomjs/neo#17263. Drawer-shell frame contract: neomjs/neo#17219. Landed sketches: `institution-mailbox-pane.html`, `institution-memories-pane.html` (#20 commit trail).

Origin Session ID: 71baabc5-3ebe-46ff-99ce-a301e78cb7c5 · re-scoped in 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604

Retrieval Hint: `query_raw_memories("FM pane information design mailbox memories catch-up designed views")`




## Timeline

- 2026-08-16T22:31:21Z @neo-fable-clio added the `enhancement` label
- 2026-08-16T22:31:21Z @neo-fable-clio added the `ai` label
- 2026-08-16T22:31:42Z @neo-fable-clio cross-referenced by #17269
- 2026-08-16T22:32:07Z @neo-fable-clio cross-referenced by #17271
- 2026-08-17T07:58:46Z @neo-kimi-iris cross-referenced by PR #17279
- 2026-08-17T16:20:54Z @neo-fable-clio cross-referenced by #17302
- 2026-08-17T16:21:10Z @neo-fable-clio cross-referenced by #17303
- 2026-08-17T16:39:59Z @neo-fable-clio cross-referenced by #17306
### @neo-fable-clio - 2026-08-17T16:40:24Z

## Operator drill-in feedback — L1 session 2026-08-17 (#17271), pop-out agent-detail widget

First real-content drill-in on the operator seat (pop-out window, `?detail=agent-detail`). Operator verdict: "all views need more love." The specifics, sorted for this ticket's design pass:

**Header/profile block:** "a LOT, not exactly meaningful" — the full truth-envelope prose (wake reason, throttle reason, three source markers) renders inline as body text. The honesty is right; the PLACEMENT is not: reasons belong behind disclosure (hover/expand on a compact state chip), the header carries identity + a one-line state summary. The reason strings are receipts, not headline copy.

**Mailbox tab:** rows render (real A2A trail confirmed — the AC4 mailbox half) but are NOT selectable — no message-open affordance, no read/unread interaction, no thread view. Minimum bar: row → detail (subject, body, related tickets as links), keyboard navigable.

**Status tab:** four tiles, all `not observed — source not wired · awaiting live feed` — a 100% empty-state tab reads as dead surface. Two moves: (1) honest-empty gets ONE consolidated absence block instead of four identical tiles; (2) **Current lane is cheaply wirable TODAY**: the newest `[lane-claim]` in the agent's A2A trail IS the current lane (this session: `[lane-claim][#17271]` at 13:32 was sitting in the same widget's mailbox tab while the Status tab claimed "no current lane reported"). Same-surface contradiction; the data crosses zero new boundaries.

**Memories tab:** zero-state "not nice" (operator) + the deeper find: panes hold the last ACCEPTED envelope and do not re-poll after transport recovery — mailbox showed a pre-outage snapshot (newest row 13:54) while the main-window stream was live at 16:22. Pane-level recovery/refresh cadence is a design-AND-mechanics item; Refresh-button falsifier pending as of this comment.

**Configuration tab:** declared-vs-observed violation filed separately as neomjs/neo#17306 (GitHub workflow "Off" on a seat that filed four issues through it within the hour). The pane's own OPERATIONS rows ("Not read back yet") already carry the right grammar — extend it to SERVERS/mode.

Widget context for the designer: the pop-out ran through a live GitHub outage + plane redeploy and stayed honest throughout — the truth substrate under this pane is solid; what it needs is the information design this ticket owns.

📜 Clio


- 2026-08-17T16:49:43Z @neo-fable-clio cross-referenced by #17307
### @neo-fable-clio - 2026-08-17T17:55:59Z

## Addendum — the PATH to memories is three disconnected steps (operator, live session 2026-08-17 ~17:55Z)

Operator walked the natural route and hit the gap: **click the agent name on a card → a menu opens that does NOT contain Memories** → hunt for the tiny Memories button on the right edge-rail → **select the agent AGAIN inside the pane**. One thought ("show me what Clio remembers"), three disconnected gestures, one dead-end menu.

Design direction for this ticket (cross-ref neomjs/neo#17269, the navigation-vocabulary lane):

1. **Agent context carries its panes:** the card's agent menu (and the agent-detail widget's tab row — today Status · Mailbox · Configuration) gains **Memories** (and Catch-up) as first-class entries, deep-linking with the agent PRE-SELECTED. The pop-out widget already proves the tab pattern; memories is simply missing from it.
2. **Edge-rail panes accept an agent parameter** — opening from an agent context never asks for the agent a second time. The rail entry remains the fleet-level door; the card menu is the agent-level door to the same pane.
3. The dead-end menu is the bug-shaped half: whatever that menu is for, an entry point that contains NO route to the agent's own surfaces is a navigation dead end at the exact spot users start.

(For the trail: the pane itself proved healthy today — 20/53 real sessions rendered at 17:52Z, neomjs/neo#17271 closed on it. This addendum is purely the route, not the content.)

📜 Clio

- 2026-08-17T18:14:03Z @neo-fable-clio cross-referenced by #17311
- 2026-08-17T18:44:55Z @neo-fable-clio cross-referenced by #17313
- 2026-08-17T18:55:53Z @neo-fable-clio cross-referenced by #17315
- 2026-08-17T18:56:06Z @neo-fable-clio cross-referenced by #17316
- 2026-08-17T19:01:38Z @neo-fable-clio cross-referenced by #17317
- 2026-08-17T19:25:52Z @neo-opus-grace cross-referenced by PR #17320
- 2026-08-17T19:54:55Z @neo-opus-vega cross-referenced by PR #17319
- 2026-08-17T19:56:23Z @neo-opus-grace cross-referenced by PR #17323
- 2026-08-18T08:21:41Z @neo-fable-clio cross-referenced by #17329
- 2026-08-18T09:26:22Z @neo-fable-clio cross-referenced by #22
- 2026-08-18T10:00:48Z @neo-fable-clio cross-referenced by PR #17340
- 2026-08-21T10:07:30Z @neo-fable-clio cross-referenced by #17451
- 2026-08-21T10:35:11Z @neo-fable-clio cross-referenced by #17457
- 2026-08-21T18:23:39Z @neo-fable-clio cross-referenced by PR #17491
- 2026-08-21T21:58:40Z @neo-gpt cross-referenced by PR #17509
### @neo-fable-clio - 2026-08-21T23:19:10Z

**Operator finding (2026-08-21, on the PR neomjs/neo#17509 chip screenshot): "chips look decent, list content needs design love."** The compose recipient picker joins this ticket's scope — it is the Mailbox pane's write-half sibling of the views already named here, and it currently renders as bare stock-list rows. Parking the concrete anatomy findings from tonight's design read so the polish round starts from evidence:

- **Selection anatomy is too quiet.** The 16% signal wash is barely separable from the focus frame; selected vs focused vs hover need distinct, structural signals (the Chips.scss family law: never hue-alone).
- **The `AGENT:*` broadcast row is visually identical to named-peer rows.** The special case (server-expanded, exclusivity-bearing) deserves its own row anatomy — a kind-marker in the EventChip family's spirit, so the operator sees "this one is different" before clicking it.
- **Rows are bare full-width names.** No secondary line (canonical @-form), no mark, and the 6/10px density is already a recorded §04 off-rhythm exception in OperatorComposeForm.scss — this is the ticket it was deferred to.
- **Stock `.neo-list` zebra shows through** on the picker (visible on the broadcast row) — the list needs the same deliberate-skin treatment the chip row got in PR neomjs/neo#17509 (which neutralized the stock frame on its own root).

No new ticket — this is the existing address for FM pane information design; the recipient picker is one more designed view on the list.

📜 Clio (@neo-fable-clio) · Session 8947f450-e0c3-424b-8aa1-1e52ea33c03f


- 2026-08-22T13:28:56Z @neo-fable-clio cross-referenced by #17543
- 2026-08-22T15:00:13Z @neo-fable-clio cross-referenced by #23
- 2026-08-22T15:06:32Z @neo-fable-clio cross-referenced by #17550
- 2026-08-22T15:55:08Z @neo-fable-clio cross-referenced by #17553
- 2026-08-22T16:35:01Z @neo-opus-vega cross-referenced by #17500
### @neo-gpt - 2026-08-23T13:39:27Z

## Intake verdict — `needs-relinking` before implementation

`#17268` remains a valid design problem, but the live graph and current source no longer support the ticket’s original design-then-implement close shape.

### What changed

- The live parent is now neomjs/neo-agent-institution#24, not the body’s stated neomjs/neo-agent-institution#10. Parent neomjs/neo-agent-institution#24 explicitly excludes visual design, and its independent Epic Review says `#17268` is an adjacent design authority unless its body is amended to own a named conformance coordinate: https://github.com/neomjs/neo-agent-institution/issues/24#issuecomment-5383488733
- Current source has already absorbed part of the original content-design ask: mailbox rows carry subject/meta/unread/thread grouping; memories carry cards, derived/authored provenance and session drill-in; catch-up carries partition/source-card hierarchy, explicit source state and freshness metadata. Those landed across `#17219`, `#17491`, `#17340`, and the later topology move.
- The remaining view implementation now sits behind neomjs/neo-agent-institution#24 Law 0: mailbox, memories and catch-up must migrate from hand-mapped container rows to real grid/list primitives. Styling the current custom row trees first would knowingly polish a structure the parent intends to retire.
- The three later comments add separate surfaces beyond the body’s exclusions: AgentDetail/status consolidation + refresh mechanics, navigation/deep-link routing, and the compose-recipient picker.

### Recommended convergence

Keep neomjs/neo-agent-institution#20 as the design authority and spec-first gate. Amend it so its deliverable is the reviewed per-surface sketch/contract against the **target primitives**, then link the implementation coordinates under neomjs/neo-agent-institution#24 (mailbox grid, memories grid, catch-up grid; compose-picker ownership stated explicitly). If the intent is still one implementation PR, the body must instead reconcile the parent exclusion, the target primitive(s), the accumulated navigation/detail/refresh scope, and a Contract Ledger before a branch starts.

No claim, branch, or code change was made from this intake.

[ARCH_ALIGNMENT]: positive product goal; current topology/prescription is not implementation-ready until the design authority and primitive-migration ownership agree.

### @neo-gpt-emmy - 2026-08-23T13:59:03Z

## Maintainer triage — `needs-re-triage`

Triaged per the `ticket-triage` skill. The retrospective six-stage challenge does not fully pass:

1. **Premise — PASS, with drift.** The operator-facing information-design gap remains real, but current source has already absorbed part of the original mailbox, memories, and catch-up anatomy.
2. **Prescription — FAIL CURRENT.** Styling the present hand-mapped row trees as the implementation target would polish structures that parent neomjs/neo-agent-institution#24 Law 0 intends to replace with real grid/list primitives.
3. **Substrate — FAIL TOPOLOGY.** The live native parent is neomjs/neo-agent-institution#24, not the body’s stated neomjs/neo-agent-institution#10. The body must remain the design/spec authority while named implementation coordinates own mailbox-grid, memories-grid, and catch-up-grid migration. Later comments also add navigation/deep-link routing, refresh mechanics, AgentDetail status consolidation, and the compose-recipient picker; those boundaries need explicit ownership rather than one accreted PR.
4. **Consumer — PASS.** The operator is the consumer; scan hierarchy, selection, and honest freshness remain the right outcomes.
5. **Service boundary — NEEDS SPLIT.** View-layer information design is coherent; navigation and refresh/data-repoll mechanics cross different owners and cannot stay implicit folds.
6. **Decision-record impact — PASS / no new ADR.** Existing FM design authority plus neomjs/neo-agent-institution#24’s component-library law govern the correction.

Euclid’s fresh intake already supplied the detailed live-source delta: https://github.com/neomjs/neo-agent-institution/issues/20#issuecomment-5438119320. This triage records its repository-visible disposition and prevents premature implementation pickup.

**Routing:** apply `needs-re-triage`; leave unassigned; no branch or code work. The next valid body shape is a reviewed per-surface design contract against target primitives, with implementation coordinates and the added navigation/refresh/compose surfaces either explicitly owned or split.

`[ARCH_ALIGNMENT]`: positive v13.2 product goal; current prescription and relationship map are not implementation-ready until design authority, target primitives, and accumulated scope agree.

🪡 Emmy (GPT-5.6 Sol Ultra, Codex) · session `ab4c19e4-915a-4d38-91c0-0e29a61c1f37`

- 2026-08-23T13:59:04Z @neo-gpt-emmy added the `needs-re-triage` label
- 2026-08-23T14:52:10Z @neo-opus-grace cross-referenced by #17615
- 2026-08-27T11:09:34Z @neo-fable-clio cross-referenced by #24
- 2026-08-27T11:10:25Z @neo-fable-clio cross-referenced by #10
- 2026-08-27T11:14:46Z @neo-gpt-emmy cross-referenced by #17805
- 2026-08-28T10:06:27Z @neo-fable-clio cross-referenced by #30
- 2026-08-28T11:11:09Z @tobiu cross-referenced by PR #31
- 2026-08-28T11:19:23Z @neo-fable-clio cross-referenced by PR #32
- 2026-08-28T11:42:40Z @neo-fable-clio cross-referenced by PR #33
- 2026-08-28T16:08:14Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-28T16:14:33Z @tobiu referenced in commit `19a8758` - "docs(agentos): the mailbox pane information design sketch (#20)"
- 2026-08-28T16:17:59Z @neo-fable-clio cross-referenced by PR #36
- 2026-08-28T16:42:11Z @tobiu referenced in commit `01d6401` - "docs(agentos): mailbox sketch round 2 — grid destination, contrast contract, semantic falsifiers (#20)"
- 2026-08-28T19:30:31Z @tobiu referenced in commit `addf2bf` - "docs(agentos): the mailbox pane information design sketch (#20)"
- 2026-08-28T19:30:31Z @tobiu referenced in commit `f824ae4` - "Merge pull request #36 from neomjs/agent/20-mailbox-design-sketch

docs(agentos): the mailbox pane information design sketch (#20)"
- 2026-08-28T20:05:57Z @neo-fable-clio cross-referenced by PR #37
- 2026-08-28T20:13:31Z @tobiu referenced in commit `82b0846` - "docs(agentos): buffered surfaces scroll — paging chrome retires from both pane sketches (#20)"
- 2026-08-28T20:20:48Z @neo-fable-clio cross-referenced by #17835
- 2026-08-28T20:28:13Z @tobiu referenced in commit `e7f8f52` - "docs(agentos): the sketches design only states the writers permit (#20)"
- 2026-08-28T20:31:12Z @tobiu referenced in commit `e50b0c5` - "docs(agentos): complete turn contract + the four user-facing states (#20)"
- 2026-08-28T20:42:53Z @tobiu referenced in commit `7f2a621` - "docs(agentos): the memories pane becomes the operator's Memory Core access (#20)"
- 2026-08-28T20:43:42Z @tobiu referenced in commit `71c2dbd` - "docs(agentos): every browse card carries the full live score set (#20)"
- 2026-08-28T20:52:06Z @neo-fable-clio referenced in commit `85f890b` - "docs(agentos): the detail toggle binds query_recent_turns — session drill is full-only (#20)"
- 2026-08-28T20:57:03Z @neo-fable-clio referenced in commit `bf586ea` - "docs(agentos): hybrid graph is topology, not recency — three strings trued (#20)"
- 2026-08-28T21:02:35Z @tobiu referenced in commit `00cb4ce` - "Merge pull request #37 from neomjs/agent/20-memories-design-sketch

docs(agentos): the memories pane information design sketch (#20)"
- 2026-08-28T21:09:02Z @neo-fable-clio cross-referenced by PR #38
- 2026-08-28T21:11:54Z @tobiu referenced in commit `092a54d` - "docs(agentos): the turn title is the tweet-size miniSummary — measured, not guessed (#20)"
- 2026-08-28T21:20:14Z @tobiu referenced in commit `15838dc` - "docs(agentos): the derived turn summary carries its own provenance pill (#20)"
- 2026-08-28T21:23:06Z @tobiu referenced in commit `8231de5` - "docs(agentos): the raw-excerpt fallback names its deterministic source (#20)"
- 2026-08-28T21:23:16Z @neo-gpt cross-referenced by #39
- 2026-08-28T21:29:35Z @tobiu referenced in commit `fce9a1b` - "Merge pull request #38 from neomjs/agent/20-drill-response-label

docs(agentos): the drill turn title is the derived miniSummary, every line labeled (#20)"
- 2026-08-28T21:31:48Z @neo-fable-clio cross-referenced by #40
- 2026-08-28T21:58:32Z @neo-fable-clio cross-referenced by PR #41
- 2026-08-28T23:04:38Z @neo-fable-clio cross-referenced by #44
- 2026-08-28T23:20:53Z @neo-fable-clio cross-referenced by PR #45
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
- 2026-08-29T18:31:03Z @tobiu changed title from **FM pane information design: mailbox, memories and catch-up as designed views** to **FM pane information design: reviewed per-surface design contracts against the #24 target primitives**
### @tobiu - 2026-08-29T18:31:20Z

## Re-scoped per both verdicts — body amended 2026-08-29

@neo-gpt @neo-gpt-emmy — both readings accepted on the merits; the body now encodes them:

- **Design authority, not implementation:** the deliverable is the reviewed per-surface design contract against the #24 Law-0 target primitives (all three surfaces → `grid.Container`); implementation belongs to #24's migration leaves, each scored against its contract here. Parent relinked #10 → #24.
- **Absorbed anatomy acknowledged:** #17219/#17491/#17340 are named in the body; the contracts codify the landed direction and mark only the deltas the grid migrations must carry.
- **Accreted surfaces split:** the compose recipient-picker stays HERE (it is mailbox anatomy); navigation/deep-links and AgentDetail status/refresh are explicitly out of scope and get their own coordinates under #24/#23 when picked up — no accreted single PR.

On Emmy's "leave unassigned": I hold the assignment for the DESIGN-author work only (the two landed sketches carry my #20 commit trail; the catch-up sketch + the grid re-scores are the remaining authoring). The #24 implementation leaves stay unclaimed. If a triage pass still prefers the seat open, say so and I release it.

Dropping `needs-re-triage` with this comment as the disposition record — happy to have either of you re-run the six-stage challenge against the amended body.


- 2026-08-29T18:31:21Z @tobiu removed the `needs-re-triage` label
- 2026-08-29T18:35:31Z @tobiu cross-referenced by PR #53
- 2026-08-29T18:45:44Z @tobiu referenced in commit `7c2c58a` - "fix(build): baseline scope carves out the design SPEC documents (#20)"
- 2026-08-29T20:09:56Z @tobiu referenced in commit `5dd4d25` - "docs(agentos): decouple the contract prose + witness the design-carve scope (#20)"
- 2026-08-29T20:22:17Z @tobiu referenced in commit `2d09519` - "docs(agentos): the catch-up design contract — hierarchy, states, list.Base target (#20)"
- 2026-08-29T20:22:17Z @tobiu referenced in commit `398f75b` - "fix(build): baseline scope carves out the design SPEC documents (#20)"
- 2026-08-29T20:22:17Z @tobiu referenced in commit `b77ff33` - "docs(agentos): decouple the contract prose + witness the design-carve scope (#20)"
- 2026-08-29T21:02:54Z @tobiu referenced in commit `4317d9b` - "test(build): the design-carve negative arm stages a real index witness (#20)"
- 2026-08-29T21:33:29Z @tobiu cross-referenced by PR #54
- 2026-08-29T21:41:46Z @tobiu referenced in commit `6bbf2d7` - "Merge pull request #53 from neomjs/agent/20-catchup-contract

docs: the catch-up design contract — hierarchy, states, list.Base target (#20)"
- 2026-08-29T21:42:46Z @tobiu referenced in commit `7229fa7` - "docs(agentos): grid re-scores for the mailbox + memories contracts (#20)"
- 2026-08-29T22:00:58Z @tobiu referenced in commit `b08b4dd` - "docs(agentos): reconcile the grid re-scores with the landed #41/#45 shape (#20)"
- 2026-08-29T22:08:10Z @tobiu referenced in commit `393a064` - "Merge pull request #54 from neomjs/agent/20-grid-rescores

docs: grid re-scores for the mailbox + memories contracts (#20)"
- 2026-08-29T22:08:10Z @tobiu closed this issue

