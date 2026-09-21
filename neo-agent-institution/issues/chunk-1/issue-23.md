---
id: 23
title: 'Cockpit header and agent detail: information architecture and responsiveness'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-08-22T15:00:12Z'
updatedAt: '2026-09-04T16:40:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/23'
author: neo-fable-clio
commentsCount: 5
parentIssue: 10
subIssues:
  - '[x] 59 Cockpit bar IA: status-word pills, state block, declared collapse order'
  - '[x] 60 Agent detail rail IA: identity row, one state ledger, capacity gating, tab-seam pop-out'
subIssuesCompleted: 2
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T16:40:47Z'
---
# Cockpit header and agent detail: information architecture and responsiveness

## Context

Operator inspection, 2026-08-22 ~14:45Z, with screenshots and an explicit brief: "information architecture and responsiveness for the header and agent profile view need severe work. … it is not 'make chips smaller' — which items should be in there? in which order? the ~730px wide error label between buttons feels unprofessional. the 2nd wake label gets cut off. on big resolutions both labels could be below each other (vertical space is there); smaller widths matter too. challenge the content of the label messages too." Measured the same hour in the running cockpit (dev `1cd9d3fd38`).

## The Problem (measured)

**Agent detail (`.fm-agent-detail`, 280–295 px rail):**
1. The identity/status column is squeezed to **107 px** (48 px avatar + padding left), so every state line wraps mid-phrase to two lines ("wake: not / reported", "Runtime: not / wired" — six 2-line rows). The break is layout-internal: it happens at ANY rail width.
2. The **"Pop out detail" shell tool overlaps content**: the button (63 px, itself clipped to "Pop out de…") renders at x≈201 over the identity block — `elementFromPoint` at its left edge hits the name text.
3. **Three vocabularies in one flat prose stack**: liveness (`active`, `wake:`, `throttle:` — italic), wiring (`Runtime/Repository/Roster: not wired`), and a dangling provenance line (`— fleet:listAgents`) — while `.fm-freshness` already exists as THE state-pill language. Then **every axis is told twice**: each drawer section below (Thought stream / Current lane / Repository / Pull requests) spends three lines re-stating the same fact ("not observed — source not wired" + "awaiting live feed").

**Header (`.agent-top-toolbar`, 1280 px):**
4. Structure: logo 30 + label 69 + instance switcher 200 + **one unstructured 877 px middle** (view switcher + spine banner **478 px** + Reconnect + wake chip **265 px, clipped mid-word** + Start fleet) + theme 49. Ellipsis + T5 titles exist, but there is **no declared item set, order, or collapse behavior** — the banner sits between buttons, the wake chip truncates at "Failed to fe", and nothing defines the minimum honest form.
5. **The label content itself is the IA defect**: the banner packs five facts into one inline sentence (endpoint · state · consequence · instruction · shell command — `127.0.0.1:8083/fleet — Fleet server offline — showing the static roster · start it: npm run ai:fleet-server`). Remediation commands and endpoints are detail/tooltip content, not chrome.

## The Architectural Reality

- `apps/agentos/view/fleet/AgentDetail.mjs` + `resources/scss/src/apps/agentos/fleet/AgentDetail.scss` (the S1 `.fm-freshness` vocabulary lives there and in MailboxPane — one language, currently unused by the header stack of the same pane).
- The spine banner (`.fm-spine-banner`) + wake telltale render in the cockpit control bar (`FleetCockpit.mjs` `syncControlBar` region); both already carry honest T5 titles.
- Container-query infrastructure exists (`FleetCockpit.scss`: `container-name: fm-cockpit; container-type: inline-size`) and is unused by these surfaces.
- Prior art: neomjs/neo#15848 (CLOSED — witnessed the 571–970 px header band once), neomjs/neo#17543 / PR neomjs/neo#17544 (the token-layer pass these surfaces now sit on), the drawer-shell mount-agnostic pane contract.

## The Fix (design-first — AC-1 sketch gate, the neomjs/neo#17329 precedent)

Answer the operator's questions in a §04 sketch BEFORE code:

1. **Header item set + order (content-first):** exactly four classes — identity/context (logo · instance), view (Overview/Focus/Review), state (TWO axes: fleet connection, wake channel — as status words/pills, never prose sentences), primary action (Start fleet; Reconnect contextual when offline). Everything else (endpoint, consequence, remediation command) moves to titles/popover/detail surfaces.
2. **Label content rewrite:** banner → `● Fleet offline · Reconnect` (endpoint + consequence in the title; the `npm run` command in the detail surface); wake → a `wake off`-class pill with the failure reason in the title (T5 pattern stays).
3. **Responsive plan with a declared collapse order** (container queries): wide → the two state lines stacked vertically inside the 50 px band (the operator's observation: the vertical space is there); mid → two pills side by side; narrow → two dots with titles. State sits right-aligned before the actions, never between buttons.
4. **Detail identity header** = one grid row (avatar ≤32 px, name + family chip, handle); the pop-out toggle moves into pane chrome (icon-only under a container width), never overlapping content.
5. **Detail state = ONE label/value ledger** in the `.fm-freshness` vocabulary (axis · state word, labels `nowrap`), provenance as pill/title — each fact told once; the drawer sections keep only their own content and a one-line section provenance pill.
6. Tokens/rhythm §04 throughout; zero CSS-in-JS; both themes.

Decision Record impact: none — aligned-with the §04 bar and neomjs/neo#17269's chrome tier.

## Acceptance Criteria

- [ ] AC-1: §04 design sketch (header + detail, incl. the label-content rewrite and the collapse order) posted and reviewed BEFORE implementation (operator or peer §04 read).
- [ ] Header renders the four item classes in the declared order; state never sits between action buttons; at container widths ≥ wide the two state lines stack vertically; the declared collapse order holds at 1280 / 1024 / 900 / 730 with computed receipts — no mid-word clipping anywhere; full truth stays reachable via T5 titles.
- [ ] No header label carries an endpoint or shell command inline; remediation lives in title/popover/detail.
- [ ] Agent detail: no state line wraps mid-phrase at rail widths 240–360 px; no control overlaps content (computed receipts); the pop-out toggle lives in pane chrome.
- [ ] Every liveness/wiring axis renders once, in the `.fm-freshness` vocabulary; the drawer sections drop the per-section "not observed" boilerplate in favor of a one-line provenance pill.
- [ ] Token-governed in both themes; `npm run check-theme-surfaces` green.

## Out of Scope

South panes' content design (#17268) · the boot-truth fix (#17547) · new data sources or verbs · the release video (#17266, sequenced after neomjs/neo-agent-institution#20 per the 2026-08-22 status answer).

## Related

Epic neomjs/neo-agent-institution#10 (parent) · neomjs/neo-agent-institution#20 · neomjs/neo-agent-institution#13 · neomjs/neo#17543 / PR neomjs/neo#17544 · neomjs/neo-agent-institution#9 · neomjs/neo#17547 · neomjs/neo#15848 (prior header-band witness, closed)

Live latest-open sweep: latest 20 open issues checked 2026-08-22T14:56Z — no equivalent (#17547 = boot truth, adjacent). A2A window (10 newest, all read-states, 14:57Z): no competing claim. Local exact sweep: 5 hits — neomjs/neo#15848/#14310/#14771 CLOSED, neomjs/neo-agent-institution#9 (epic, related), neomjs/neo#15097 (unrelated).

Origin Session ID: 14acab5a-4b6c-4987-91c7-f683e39baa55

Retrieval Hint: `query_raw_memories("cockpit header information architecture collapse order agent detail freshness ledger overlap pop out")`

## Timeline

- 2026-08-22T15:00:13Z @neo-fable-clio added the `enhancement` label
- 2026-08-22T15:00:13Z @neo-fable-clio added the `design` label
- 2026-08-22T15:00:13Z @neo-fable-clio added the `ai` label
- 2026-08-22T15:00:14Z @neo-fable-clio added the `agent-os` label
- 2026-08-22T15:00:21Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-08-22T15:06:32Z @neo-fable-clio cross-referenced by #17550
- 2026-08-22T15:53:07Z @neo-fable-clio unassigned from @neo-fable-clio
- 2026-08-22T15:55:08Z @neo-fable-clio cross-referenced by #17553
- 2026-08-23T01:01:13Z @neo-gpt-emmy cross-referenced by #24
- 2026-08-26T00:05:13Z @neo-preview assigned to @neo-preview
### @neo-preview - 2026-08-26T00:06:56Z

## AC-1 — §04 design sketch: header item set, label rewrite, collapse order, agent-detail ledger

Spec-first gate deliverable. Every symbol below exists at current dev; nothing new is invented except one pill class reusing an existing register.

### 1. Header item set & order (`fm-cockpit-bar`, four classes, declared)

| order | class | items | notes |
|---|---|---|---|
| 1 identity/context | logo · instance switcher | unchanged positions | switcher collapses to icon at narrow |
| 2 view | preset switcher (`fm-preset-button` group) + `fm-preset-error` | unchanged | |
| 3 state | NEW `fm-state-block` — TWO pills, right-aligned as ONE group: `fm-fleet-pill` + `fm-wake-pill` | **never between buttons**; replaces spine-banner-as-prose AND the wake chip | |
| 4 primary action | `Reconnect` (contextual — visible only when fleet pill is offline) · `Start fleet` · theme toggle | Reconnect keeps its existing handler (`reconnectFleet`) | |

**Pill content** (`.fm-state-pill`, register = `.fm-freshness`):
- Fleet: `● Fleet offline` / `● Fleet live` — T5 title carries endpoint + consequence + start command (the current banner's five facts move HERE, verbatim text preserved in the title)
- Wake: `wake off` / `wake live` — T5 title carries the failure reason (current chip's clipped sentence becomes reachable truth)

The zero-nominal-pixels rule is preserved honestly: a fully live spine renders pills in their live state (`Fleet live · wake live`) instead of hiding — two small truths beat one hidden sentence, and the operator asked for state that is always present.

### 2. Collapse order (container queries on the existing `fm-cockpit` inline-size context)

| band | state block | other changes |
|---|---|---|
| ≥1180 wide | two pills stacked vertically inside the ~50 px bar band (the operator's "vertical space is there") | full switchers |
| 900–1179 mid | two pills side-by-side, single-line | instance switcher → icon+name |
| <900 narrow | pills collapse to color dots (10 px), titles carry all truth | view switcher → dropdown; labels drop |

Declared minimum honest form at 730 px: logo-dot · view-dropdown · [2 dots] · Start · theme. Nothing clips mid-word because nothing renders a word it cannot fit.

### 3. Agent detail (`.fm-agent-detail` rail)

- **Identity row** = one grid line: avatar 32px · name + family chip (one line, `nowrap`) · handle underneath. The 107px-squeeze wrap dies because identity stops sharing a column with state.
- **Pop-out toggle** moves to pane chrome top-right, icon-only (`window-restore` icon) under a container width — the current 63px text button over content is deleted. `detail-window-toggle` reference and `syncControlBar()` wiring stay; only its host and label change.
- **State = ONE `.fm-freshness` ledger**: rows `liveness · wake · throttle · runtime · repository · roster` — axis label left (nowrap), ONE state word right. Provenance (`— fleet:listAgents`) becomes a footer pill. Drawer sections drop their three-line "not observed" boilerplate for a one-line section provenance pill.
- Rail container query already precedented (`detail/Container.scss` max-width 315px); extends to the 240–360px AC band.

### 4. Implementation surface (unchanged owners)

`cockpit/Container.mjs syncControlBar/buildWorkspaceItems` (state block + pill sync) · `cockpit/Container.scss` (collapse queries) · `AgentDetail.mjs` + `fleet/detail/Container.scss` (ledger + chrome move). Zero CSS-in-JS; both themes via tokens; `check-theme-surfaces` green.

### Open question for the §04 reader
Keep or kill the live-state pills? My sketch says keep (always-present truth); the alternative is hide-when-live (today's behavior). Decision needed before implementation — it changes the bar's nominal pixel cost.


- 2026-08-27T11:09:28Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-27T11:09:28Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T11:09:29Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T11:09:29Z @neo-gpt-emmy added the `design` label
- 2026-08-27T11:14:46Z @neo-gpt-emmy cross-referenced by #17805
- 2026-08-28T10:06:27Z @neo-fable-clio cross-referenced by #30
- 2026-08-28T11:11:09Z @tobiu cross-referenced by PR #31
- 2026-08-28T11:19:23Z @neo-fable-clio cross-referenced by PR #32
- 2026-08-29T11:42:35Z @tobiu unassigned from @neo-preview
- 2026-08-29T18:31:04Z @tobiu cross-referenced by #20
- 2026-08-29T22:53:23Z @neo-fable-clio cross-referenced by PR #57
- 2026-08-29T23:15:01Z @neo-fable-clio cross-referenced by PR #58
### @neo-fable-clio - 2026-08-29T23:15:18Z

## AC-1 discharged + the sketch→consumer binding (the no-orphan-specs rule)

The operator flagged a real failure class: design specs committed without a consuming coordinate get ignored forever. Recording the binding explicitly so this sketch cannot join that class:

- **AC-1 (the §04 sketch):** `apps/agentos/design/institution-header-detail-ia.html`, landed via PR #57 after the operator's live read ("definitely a LOT better. please drive it") — that read IS the AC-1 gate.
- **Cut A (cockpit bar) — PR #58, open:** consumes the sketch's header decisions (state block, status-word pills, label rewrite, declared collapse order). The sketch's header sections have their consumer.
- **Cut B (agent detail rail) — NEXT, this lane, not yet a PR:** consumes the detail sections (identity grid row, one freshness ledger, pop-out onto the tab header action seam, throttle→capacity rename + source-gating). I hold the lane until it lands; if this lane is ever released with Cut B unshipped, whoever picks #23 up inherits the sketch's detail sections as the SPEC of that cut — they are not optional prose.
- **Explicitly NOT consumed here** (named owners, not orphans): the typed connection-state pill vocabulary (#18/#15 feed the pill TEXT when they land) and the §04 chip-family unification on the two state chips (the #24 conformance leaf).

#23 stays open until Cut B lands; its remaining ACs are the checklist.

📜 Clio (Fable 5, Claude Code) · session 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604


- 2026-08-29T23:30:59Z @tobiu referenced in commit `2e7a2e6` - "Merge pull request #57 from neomjs/agent/23-header-detail-ia-sketch

docs(agentos): header + agent-detail information-architecture sketch (#23)"
- 2026-08-29T23:34:22Z @neo-fable-clio cross-referenced by #59
- 2026-08-29T23:34:23Z @neo-fable-clio cross-referenced by #60
### @neo-fable-clio - 2026-08-29T23:35:44Z

## Structure correction (operator rule): every cut resolves its own sub-ticket

The operator ruled on the open cockpit-bar PR: a PR must always RESOLVE a ticket + its ACs — Refs-only is forbidden (engine-CI law, applied here). The binding comment above is amended accordingly:

- **#59** — Cockpit bar IA (sub of this ticket): resolved by PR #58 (body flipped from `Refs #23` to `Resolves #59`).
- **#60** — Agent detail rail IA (sub of this ticket): resolved by the Cut-B PR (committed at `b6c476a`, opens serially once #58 merges — same lane).
- **#23** (this ticket) closes when its checklist is fully discharged across the merged subs — AC-1 (sketch) is done via PR #57 + the operator read.

📜 Clio (Fable 5, Claude Code) · session 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604


- 2026-08-29T23:56:07Z @neo-fable-clio cross-referenced by PR #61
- 2026-08-30T00:00:16Z @tobiu referenced in commit `9ab2d81` - "Merge pull request #58 from neomjs/agent/23-cockpit-bar-ia

feat(agentos): cockpit bar IA — status-word pills + state block + collapse order (#23)"
### @neo-fable-clio - 2026-08-30T00:23:43Z

## Session-sunset handover (2026-08-30, @neo-fable-clio — dark until Friday 08:00)

**State: every implementation cut is MERGED.** AC-1 sketch (PR #57, operator read), Cut A cockpit bar (#59 → PR #58), Cut B detail rail (#60 → PR #61, incl. the 64px portrait, name-edge handle alignment, and the APP-WIDE tab-header-action contract in Viewport.scss targeting the engine's `neo-toolbar-action` seam). All operator-review iterations landed inside the cuts; goldens re-captured deliberately; stamps index-true at dev `f97187a`.

**What keeps this ticket open (the honest residue):** the AC receipt rows demand COMPUTED receipts at 1280/1024/900/730 (header) and 240–360 (rail). Partially witnessed today — the 720 visual witness pins the mark regime with computed geometry, midline equality was measured live (371===371), and the unit ledgers pin one-line-height rows — but no formal four-width receipt series is recorded on this ticket. Pickup protocol: either (a) run the four-width probe (the scratch pattern: headless chromium + getBoundingClientRect over the bar/rail, ~20 lines — see the boot-heal ticket #62's evidence style) and close with the receipt table, or (b) the operator declares the live read sufficient and closes directly.

**Do NOT re-open design questions here:** label vocabulary (#18/#15 feed the pill text), chip-family unification (#24 leaf), boot-auto-heal (#62 owns it).

📜 Clio (Fable 5, Claude Code) · session 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604


- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T21:42:14Z @neo-fable-clio cross-referenced by PR #70
- 2026-09-01T22:08:09Z @neo-fable-clio cross-referenced by #10
- 2026-09-01T22:14:16Z @neo-opus-ada cross-referenced by PR #71
- 2026-09-02T15:49:48Z @neo-fable-clio cross-referenced by #85
- 2026-09-04T10:08:33Z @neo-fable-clio cross-referenced by PR #93
- 2026-09-04T16:35:20Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-04T16:40:45Z

## Closure receipts — the computed geometry the AC rows asked for (2026-09-04, dev `da4657c`, engine pin `205bc52f8a`, headless Chromium, the static roster)

Every implementation cut merged on 2026-08-30 (the §04 sketch PR #57 with the operator's live read; the cockpit bar #59 → PR #58; the detail rail #60 → PR #61). What kept this ticket open was the receipt series; here it is, from an untracked Playwright probe (viewport series, `getBoundingClientRect` / `scrollWidth` / `elementFromPoint` reads; the probe is archived in the origin session's scratchpad, not in the repo).

### Bar — the declared collapse order and the item order, computed

The `fm-cockpit` container is the viewport minus the shell rail (≈ −51 px), so the AC's four viewports land in the three designed forms: wide (> 1180) · row (≤ 1180) · marks (≤ 760).

| viewport | container | form (expected) | presets → state → actions (x edges) | state between actions? | clipping | doc overflow | T5 titles |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1280 | 1229 | **stacked** (stacked) ✓ | presets ≤351 · state 882–987 · actions ≥995 ✓ | no ✓ | none ✓ | 0 px ✓ | both ✓ |
| 1024 | 973 | **row** (row) ✓ | presets ≤351 · state 546–731 · actions ≥739 ✓ | no ✓ | none ✓ | 0 px ✓ | both ✓ |
| 900 | 849 | **row** (row) ✓ | presets ≤351 · state 422–607 · actions ≥615 ✓ | no ✓ | none ✓ | 0 px ✓ | both ✓ |
| 730 | 679 | **marks** (marks) ✓ | presets ≤351 · state 547–593 · actions ≥601 ✓ | no ✓ | none ✓ | 0 px ✓ | both ✓ |

Bar children in DOM order at every width: preset × 3 → (spacer) → `fm-bar-state` → `fm-reconnect-button` → `fm-fleet-start` — the four classes in the declared order, the state block right-aligned before the actions. Pill text is the status word only (`fleet offline` 105×19, `wake off` 72×19 at 11 px; 19×16 marks at font-size 0 in the narrow form) and the T5 titles carry the full truth at every width (fleet: "Fleet server offline — showing the static roster · start it from the n…"; wake: "wake push unavailable — wake stream disconnected (not started) — poll …"). No header label carries an endpoint or a shell command inline. Action labels: `Reconnect` / `Start fleet` shown through 900, icon-only at 730; the view labels never drop.

### Rail — the detail pane across the Review band, computed

The Review preset's inspector band is 25 % of the host, so viewports 1000–1480 give the AC's rail widths; measured with the first roster card selected.

| viewport | rail width | ledger rows one line high | avatar | name | control over the identity header? | name hit-test | rail overflow | pane-chrome actions |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1480 | 357 px | all 5 ✓ (status 15/14, wake 15/14, runtime 15/14, repository 15/14, roster 15/14; lh 15) | 64 px | 187 px | none ✓ | identity ✓ | 0 px ✓ | tabs `Status` 61×30, `Configuration` 114×30; `fm-detail-window-toggle.neo-toolbar-action` 24×24 at the header's right edge |
| 1280 | 307 px | all 5 ✓ (same heights) | 64 px | 137 px | none ✓ | identity ✓ | 0 px ✓ | same seam |
| 1100 | 262 px | all 5 ✓ (same heights) | 64 px | 92 px | none ✓ | identity ✓ | 0 px ✓ | same seam |
| 1000 | 237 px | all 5 ✓ (same heights) | 64 px | 67 px | none ✓ | identity ✓ | 0 px ✓ | same seam |

Every liveness/wiring axis renders once (`status · wake · runtime · repository · roster`, axis word + `.fm-freshness` pill, each row one line-height of 15 px at every width); the name is the only ellipsizing line (its width follows the rail, never a wrap); the pop-out toggle lives in the tab header's action seam (`neo-toolbar-action`), 24×24, and no button intersects the identity header — `elementFromPoint` on the name lands in `.fm-detail-identity` at every width. The drawer sections carry one provenance pill each (`not observed — source not wired`), the per-section boilerplate is gone (the 1280 Review golden shows the form).

### The AC checklist, discharged

- AC-1 §04 sketch posted and read — PR #57 + the operator's live read (2026-08-29). ✓
- Header: four item classes in the declared order; state never between action buttons; ≥ wide the two state lines stack; the collapse order holds at 1280 / 1024 / 900 / 730 with computed receipts; no mid-word clipping; full truth in T5 titles — the bar table. ✓
- No header label carries an endpoint or shell command inline — the pill texts and titles above. ✓
- Agent detail: no state line wraps at rail widths 240–360 (measured 237–357); no control overlaps content; the pop-out toggle lives in pane chrome — the rail table. ✓ (The avatar reads 64 px, not the sketch's "≤ 32 px": the operator's call of 2026-08-30, recorded in `detail/Container.scss` — size encodes drill depth above the 40 px card tier.)
- Every axis once in the `.fm-freshness` vocabulary; the drawer sections drop the boilerplate for one provenance pill — the rail table and the golden. ✓
- Token-governed in both themes; `npm run check-theme-surfaces` green — the script named here guards the engine's Workstation surface and left this repository with the split (it does not cover `apps/agentos`). The substitute receipt: the two stylesheets (`cockpit/Container.scss` 326 lines, `detail/Container.scss` 251 lines) read 42 and 34 `var(--…)` tokens and contain no color literal in a rule body (every `#`/`rgb(` match is a ticket number or a quoted stock value inside a comment); both skins supply the values the rules read, and the operator-mailbox NL witness exercises both themes. ✓ (as substituted)

Design questions stay where they were routed: the typed connection-state pill text (#18 / #15), the chip-family unification (#24), the boot heal (#62, merged). Closing.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

- 2026-09-04T16:40:48Z @neo-fable-clio closed this issue
- 2026-09-04T17:33:03Z @neo-fable-clio cross-referenced by #107

