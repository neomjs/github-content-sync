---
id: 269
title: Map old tickets onto current Neo reality in ticket-intake and ticket-triage
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-04T14:39:10Z'
updatedAt: '2026-11-04T14:39:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/269'
author: neo-gemini-maryam
commentsCount: 1
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
# Map old tickets onto current Neo reality in ticket-intake and ticket-triage

## Context

`Serves:` **none** — standalone skill-substrate refinement in `ticket-intake` and `ticket-triage` prompted by operator review of `neomjs/neo#20252` (`#4723`) and `neomjs/neo#20282`.

Design authority: "These bands describe close-inactive automation risk only. They are NOT evidence that a ticket is architecturally current." (`.agents/skills/ticket-intake/references/successor-risk-audit.md:8`)

During review of Engine PR `neomjs/neo#20252` (resolving `#4723`, filed 2023-07-23), `@tobiu` highlighted a recurring structural gap when maintainers pick up or triage older tickets where `10k+` commits have landed since creation:
- An old ticket's **core intent** can remain completely valid (e.g. `#4723` adding `minSize` to `Neo.component.Splitter`), while its **surrounding architecture** has evolved dramatically (`Neo.dashboard.DockSplitter` extending `Splitter` for multi-sibling dock layouts where 2+ stacked containers touch one vertical splitter, and `Neo.main.draggable.Resize` adding conserved-pair `counterTargetId` resizing alongside directional drag).
- Storing this mental model only in an individual seat's `MEMORY.md` does not protect other peers picking up or triaging legacy backlog tickets.

## The Problem

Both `.agents/skills/ticket-intake/` and `.agents/skills/ticket-triage/` currently treat ticket age and historical drift as a binary **alive (`valid-as-written`) vs. dead (`already-resolved` / `superseded` / `needs-re-triage`)** test against the originally named file:

1. **`ticket-intake` (`.agents/skills/ticket-intake/references/successor-risk-audit.md` & `ticket-intake-workflow.md`):**
   - `successor-risk-audit.md` defines `§1.1 Short-Horizon Currency Check` (for tickets hours/days old) and checks whether older tickets are `already-resolved`, `superseded`, or conflict with an ADR (`§2–§3`).
   - When an older ticket is *not* resolved or superseded, an agent checks only whether the originally named class (`src/component/Splitter.mjs`) still lacks the feature, stamps `valid-as-written`, and implements it against that single class in isolation — missing **Long-Horizon Surrounding-Topology Evolution** (new subclasses via `get_class_hierarchy`, layout adapters, main-thread addons, or multi-sibling consumers introduced across the intervening `10k+` commits).
2. **`ticket-triage` (`.agents/skills/ticket-triage/references/ticket-triage-workflow.md`):**
   - `§3 Step 1 (Retrospective Six-Stage Challenge)` treats any Stage 2 (`Prescription`) or Stage 3 (`Substrate`) mismatch as a halt (`needs-re-triage`), assuming the original author should clarify (`§4`).
   - On an older ticket where `10k+` commits landed since `createdAt`, the **Premise / intent** is often still valid while the historical prescription predates newer subclasses, adapters, or engine idioms. Halting with `needs-re-triage` stalls valid backlog work; applying labels without recording how the surrounding topology moved leaves `ticket-intake` blind.

## Existing-Enforcement Sufficiency Audit (`substrate-sufficiency-audit.md`)

- **Layer 4 (CI) Checks:** `lint-skill-corpus.mjs` and `check-substrate-size.mjs` validate skill structure and byte budgets, not intake/triage surrounding-topology sweeps.
- **Layer 1 (`AGENTS.md`) Invariants:** `AGENTS.md §verify_before_assert` mandates falsifying tool calls before assertions, and `§edge_case_triggers` mandates reading 1–2 siblings for `apps/**`, neither of which specifies mapping old tickets with valid intent onto evolved subclasses/consumers (`10k+` commits) during intake/triage.
- **Existing Skill Logic:** `successor-risk-audit.md` covers `§1.1 Short-Horizon Currency Check`, `§2 Missing Close-Link Sweep`, and `§3 ADR Successor-Risk Branch`, but has no long-horizon surrounding-topology check for valid-intent older tickets; `ticket-triage-workflow.md §3 Step 1` has no distinction between a false premise and a valid historical intent whose surrounding substrate evolved.

## The Architectural Reality

- `.agents/skills/ticket-intake/references/successor-risk-audit.md` (1,015 bytes, conditional World-Atlas payload loaded from `ticket-intake-workflow.md §1 Step 7.5`) owns age and successor-risk mechanics.
- `.agents/skills/ticket-intake/references/ticket-intake-workflow.md` (16,743 bytes, workflow Map) routes `§0` (*"accept and sharpen"*), `§1 Step 7.5`, and `§1 Step 8` (`Ticket Reality Classification`).
- `.agents/skills/ticket-triage/references/ticket-triage-workflow.md` (6,481 bytes, workflow Map) owns `§2 Pre-Triage Context Pull` and `§3 Step 1 Retrospective Six-Stage Challenge`.
- Per `ADR 0007` and `AGENTS.md §self_evolving_systems` (Accretion Defense), `.agents/skills/**/SKILL.md` routers must not grow, and net byte delta across `.agents/skills/**/*.md` should remain `<= 0 bytes` by tightening adjacent verbose phrasing in `ticket-intake-workflow.md` and `ticket-triage-workflow.md`.

## The Fix

1. **`.agents/skills/ticket-intake/references/successor-risk-audit.md` (`§1.2 Long-Horizon Surrounding-Topology Check`):**
   - Add `### 1.2 Long-Horizon Surrounding-Topology Check (10k+ Commits Reality Mapping)` for older tickets whose core intent remains valid (`not already-resolved` / `not superseded`):
     - Sweep current subclasses, adapters, and consumers of the target surface (`get_class_hierarchy`, `grep` for `extends <Class>` / `ntype` / addon callers).
     - Evaluate (a) **downstream compatibility** (whether base-class changes affect specialized subclasses/multi-sibling consumers such as `DockSplitter`) and (b) **upstream generalization vs. follow-up split** (whether custom logic in a newer subclass belongs in the base class now or in a linked follow-up ticket).
     - Route via `ticket-intake-workflow.md §0` **accept and sharpen** (recording the current-reality mapping in the intake record/comment and filing any out-of-scope unification follow-up before branching).
2. **`.agents/skills/ticket-intake/references/ticket-intake-workflow.md` (`§1 Step 7.5` & `§1 Step 8`):**
   - Add a 1-line pointer in `§1 Step 7.5` / `§1 Step 8` requiring `successor-risk-audit.md §1.2` surrounding-topology mapping before `valid-as-written` on older tickets, while tightening verbose prose in `ticket-intake-workflow.md` so the file net-shrinks.
3. **`.agents/skills/ticket-triage/references/ticket-triage-workflow.md` (`§3 Step 1` & `§4`):**
   - Distinguish **invalid premise** (`needs-re-triage` halt) from **valid historical intent with evolved surrounding reality (`10k+` commits)**: when an older ticket's core intent is valid but its surrounding classes/consumers evolved since `createdAt`, re-anchor the current owning substrate and subclasses in the triage comment (`§6`) and apply current labels rather than halting on `needs-re-triage`, while tightening verbose prose so `ticket-triage-workflow.md` net-shrinks.
4. **Package version bump:** Bump `version` in `package.json` and `package-lock.json` (`0.1.85` -> `0.1.86`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `.agents/skills/ticket-intake/references/successor-risk-audit.md` (`§1.2`) | Operator direction on `neomjs/neo#20252` (`#4723`) + `ADR 0007` | Requires long-horizon surrounding-topology mapping (subclasses, adapters, multi-sibling/addon consumers via `get_class_hierarchy` / `grep`) when an older ticket's core intent remains valid | If surrounding evolution makes part of the unification out of scope, sharpen scope and file a linked follow-up ticket (`#20282` pattern) rather than silently diverging | Yes (skill payload) | `npm run lint` + `npm test` in `neomjs/neo-agent-skills` |
| `.agents/skills/ticket-intake/references/ticket-intake-workflow.md` (`§1.7.5`, `§1.8`) | `ADR 0007` + `create-skill` Map/Atlas split | Links `successor-risk-audit.md §1.2` into Step 7.5 and Step 8 `Currency / successor-risk evidence` with net-negative file byte delta | If surrounding prescription needs narrowing beyond a comment/follow-up split, route to `needs-narrowing` | Yes (skill workflow) | `npm run lint` + `npm test` + byte-delta check |
| `.agents/skills/ticket-triage/references/ticket-triage-workflow.md` (`§3 Step 1`, `§4`) | `ticket-triage` + `neomjs/neo-agent-skills#52` | Distinguishes false premise (`needs-re-triage`) from valid old intent whose surrounding architecture evolved across `10k+` commits; re-anchors current substrate/subclasses in the triage comment and applies labels | If the core premise itself is obsolete or superseded, halt/reject per existing Stage 1 rule | Yes (skill workflow) | `npm run lint` + `npm test` + byte-delta check |

Decision Record impact: aligned-with `neomjs/neo#adr:agents-md-compaction-taxonomy` (`ADR 0007`) and `neomjs/neo#adr:skill-anatomy-and-authoring-contract` (`ADR 0008`).

## Acceptance Criteria

- [ ] `.agents/skills/ticket-intake/references/successor-risk-audit.md` includes `§1.2 Long-Horizon Surrounding-Topology Check (10k+ Commits Reality Mapping)` covering subclass/consumer discovery (`get_class_hierarchy` / `grep`), downstream compatibility, and upstream generalization vs. follow-up ticket split (`neomjs/neo#4723` -> `#20252` / `#20282` anchor).
- [ ] `.agents/skills/ticket-intake/references/ticket-intake-workflow.md` references `§1.2` in Step 7.5 / Step 8 without increasing `SKILL.md` router bytes.
- [ ] `.agents/skills/ticket-triage/references/ticket-triage-workflow.md` distinguishes a false premise (`needs-re-triage`) from a valid older intent whose surrounding reality evolved (`10k+` commits), routing the latter to current-substrate re-anchoring in the triage comment + label application.
- [ ] Net byte delta across `.agents/skills/**/*.md` is `<= 0 bytes` (accretion defense), `version` is bumped in `package.json` + `package-lock.json`, and `npm run lint` + `npm test` pass cleanly.

## Out of Scope

- Modifying `AGENTS.md`, `SKILL.md` routers, or `skills.manifest.json` budgets.
- Implementing `neomjs/neo#20282` (`DockSplitter` / `Splitter` unification in `neomjs/neo`).

## Avoided Traps

- **Personal-memory trap:** Recording the `10k+ Commits` mental model only in seat-local `MEMORY.md` leaves all other peers unprotected during `ticket-intake` and `ticket-triage`.
- **False `needs-re-triage` trap on old tickets:** Treating an outdated historical prescription on a valid-intent 2020–2024 ticket as a `needs-re-triage` halt stalls valid backlog items waiting for historical authors instead of mapping the intent onto current Neo reality.
- **Additive bloat trap:** Adding paragraphs to workflow Maps without offsetting byte reductions violates `ADR 0007` and `maxPositiveDeltaBytes`.

## Related

- `neomjs/neo#20252` (`#4723`) — `Splitter#minSize` PR where the `DockSplitter` multi-sibling surrounding-reality question surfaced
- `neomjs/neo#20282` — Follow-up ticket to evaluate unifying `DockSplitter` multi-sibling and conserved-pair `minSize` bounds into base `Splitter` / `Resize`
- `neomjs/neo-agent-skills#52` & `#62` — Prior `ticket-intake` / `ticket-triage` / `ticket-create` sweep and label-routing refinements

## Sweeps & Ownership

- Live latest-open + closed sweep: checked latest 20 open issues in `neomjs/neo-agent-skills` (`#258` down to `#5`) + closed search (`ticket-intake OR ticket-triage OR successor-risk`, read `Out of Scope` / `Avoided Traps` on `#52`, `#57`, `#62`, `#192`) at `2026-11-04T14:38:00Z`; no equivalent or prior exclusion found.
- A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-11-04T14:36:35Z`; zero competing claims on `ticket-intake` or `ticket-triage`.
- Memory Core rationale sweep + own-assignment sweep: ran `query_raw_memories` (`ticket-intake ticket-triage old tickets surrounding reality successor-risk-audit`), `ask_knowledge_base` (`#10758`, `#19489`, `#1438`), and `gh issue list -R neomjs/neo-agent-skills --state open --assignee @me` (0 open assigned issues).
- Structure map: ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc`; existing `.agents/skills/ticket-intake/references/` and `.agents/skills/ticket-triage/references/` payloads own this concern.
- Ownership: self-assigning `@neo-gemini-maryam` for immediate implementation.

Origin Session ID: 7f1bbc40-971f-45e7-8c63-d4f2cf0f5155
Retrieval Hint: "ticket-intake ticket-triage 10k commits old tickets surrounding reality DockSplitter successor-risk-audit"


## Timeline

- 2026-11-04T14:39:11Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-04T14:39:12Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-04T14:39:12Z @neo-gemini-maryam added the `ai` label
- 2026-11-04T14:39:12Z @neo-gemini-maryam added the `architecture` label
- 2026-11-04T14:39:12Z @neo-gemini-maryam added the `model-experience` label
### @neo-gemini-maryam - 2026-11-04T14:39:30Z

### Ticket Intake (`self-authored-carve.md` Stage 2 Prescription Check)

- `Prescription checked: .agents/skills/ticket-intake/references/successor-risk-audit.md — owns the concern` (conditional World-Atlas payload for age and successor-risk mechanics; paired with 1-line routing in `.agents/skills/ticket-intake/references/ticket-intake-workflow.md` and Stage 1 retrospective disambiguation in `.agents/skills/ticket-triage/references/ticket-triage-workflow.md`).
- `ADR successor-risk: adr-aligned — artifact neomjs/neo-agent-skills#269/2026-11-04; ADR neomjs/neo#adr:agents-md-compaction-taxonomy (Accepted/2026-05-15) & neomjs/neo#adr:skill-anatomy-and-authoring-contract (Accepted/2026-05-16); evidence .agents/skills/ticket-intake/references/successor-risk-audit.md; route continue.`

- 2026-11-04T14:45:24Z @neo-gemini-maryam cross-referenced by PR #270

