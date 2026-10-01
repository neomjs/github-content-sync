---
id: 384
title: 'The cockpit projects the first-run recipe inline, never as a gate'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
  - design
assignees: []
createdAt: '2026-10-01T13:38:27Z'
updatedAt: '2026-10-01T13:38:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/384'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
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
# The cockpit projects the first-run recipe inline, never as a gate

## Context

Epic #351 (graduated from neomjs/neo#18965 on 2026-10-01), solution points 1, 2, 7 and 8: one shared recipe evaluated live; two renderers over one host-effect module, the cockpit inside the packaged vessel (#7) being the second; density as a measured target; Connect as the second door. The Discussion's OQ2 was dispositioned onto this leaf: #12's rule — *no setup wizard walls; an inline, dismissible setup card INSIDE the cockpit, never a modal gate, never a blank screen; a visible, honest progress line; first persistence → one quiet confirmation* — accepted by #12's author (`DC_kwDODSospM4BG0VZ`) with two lines kept: *dismissible* means the frame stays operable underneath (the connect fork reachable, the switcher live, every empty pane labelled), and *no wizard walls* is also a rule about the steps (each skippable-then-resumable, none blocking the frame, the recipe's projected progress IS #12's progress line).

The shipped precedent is `apps/agentos/view/PlaneSetupPanel.mjs` — the packaged shell's **Connect** card: inline, dismissible, never a gate; mounted by `ViewportController#mountPlaneSetup` only for a packaged shell with no plane configured; Electron main asks for the PAT in its own window, so no credential ever reaches the card and every line it renders is plain text. This leaf gives that card family its **Create** door: the recipe (neomjs/neo-agent-brain#679) projected into the cockpit.

Parent: #351 (sub-issue link set after creation). Upstream contracts: neomjs/neo-agent-brain#679 (the recipe's step JSON and the host-effect module), neomjs/neo-agent-brain#685 (the probe's budgets), neomjs/neo-agent-brain#686 (the preset table); ADR 0041 (the cockpit PROJECTS the record and never writes it).

## The Problem

A stranger who double-clicks the vessel today meets the Connect card and prose. There is no Create door, so the first run is a terminal exercise before the cockpit means anything, and the moment the cockpit could show progress it has nothing to project. Every earlier attempt at a setup surface in this codebase stored status, and a stored status is how the cockpit showed `● streaming` over a three-week-old row (2026-09-19) — the anti-anchor ADR 0041 is written against.

## The Architectural Reality

- **Authority split (ADR 0041, D#18965 option C rejected):** a browser page has no host authority. The vessel's main process (`harness/main.mjs`, `ipcMain`) is the host-effect caller: it imports neomjs/neo-agent-brain#679's host-effect module — the same one the CLI uses — and exposes a narrow IPC to the renderer. The renderer never runs compose, writes a file or probes a port.
- **Projection, not storage (ADR 0041 §2.3):** every step the card shows is the recipe's fresh `evaluate(target)` output for the bound target and recipe version; the record (consent, receipts) is read through main and shown as history; nothing in the renderer remembers a "completed" bit; a target or recipe-version change retires what is shown.
- **Credential boundary (shipped):** the PAT enters main's own window (`attachPlane()` precedent); the hosted preset's provider key follows the same path — the card never receives a secret value.
- **Neo idiom (apps/** gate):** the steps are a `data.Store` of `data.Model` records (`id`, `kind`, `status`, `reason`, `observedAt`, `action`), never a hand-mapped array; the view root's `state.Provider` holds the run (`runId`, `target`, `recipeVersion`, `placement`, `preset`, `density`) and the card binds to it; items are declarative; styles live in the theme's SCSS; the card consumes the token system (#13).
- **Owners of readiness** stay where they are: the deployment-state projection and the healthcheck's served identity (plane readiness), the probe (#685, budgets), the presets (#686). The card reads their JSON through main.
- **#14's TTFP instrument** fires at first persistence — the quiet confirmation IS that event.

## The Fix

1. **`AgentOS.view.setup`** — a `SetupCard` (a `Neo.container.Panel`, the `PlaneSetupPanel` family; the implementer decides with a structural pre-flight whether `PlaneSetupPanel` becomes the Connect arm of one card with two doors or a sibling under a shared base — one skin rule either way). Composition: the two doors (Create · Connect) · the step list (a `Neo.list`/`Neo.grid` over the step Store: status glyph, title, reason, one action per row) · the three questions as fields — placement (the probe's host and guest budgets rendered as numbers with the `pressure` verdict; a second plane only under "advanced"), preset (cards from #686's table: inference, dimension, workload, floor; `local-*` disabled with the reason when the probe refuses them), the PAT (a button that opens main's window) · the advanced fold · dismiss (the frame stays operable; resume from Home's doors #244 / the rail).
2. **Main-process IPC** (`harness/main.mjs`): `setup:evaluate` → the recipe's step JSON for the bound target; `setup:probe` → #685's JSON; `setup:presets` → #686's table; `setup:answer` (placement / preset) → the recipe re-evaluates; `setup:effect` (consent for one effect) → the host-effect module runs it, the record gets its receipt, the renderer gets the re-evaluated steps; `setup:credential` → opens main's credential window (never a value over IPC). Every reply is recipe JSON; every failure is a `reason` on a step, never a gate.
3. **The progress line** in the frame — #12's "visible, honest progress line" — bound to the step Store (n of m observed `ok`, the current step's reason); it never reads the record.
4. **The quiet confirmation** at first persistence (the J3 / #14 TTFP event), then the card retires itself from the primary slot and stays reachable.
5. **Density** — decisions and manual actions counted per path on completion and recorded on the epic (point 7's AC).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the Create door card | #12 (inline, dismissible, resumable); OQ2 `DC_kwDODSospM4BG0VZ` | primary inline content on a Brain-less boot; dismiss keeps the frame operable; resume from Home / the rail | no vessel (served cockpit) → the card shows the CLI's command and projects the CLI's `--json` when pasted/attached | class JSDoc + `#12` cross-ref | e2e: the OQ2 witness |
| step projection | ADR 0041 §2.3; #679's `evaluate()` | every row is a fresh observation; no stored status; unknown recipe version → mismatch shown | an IPC failure → every step `unknown` with the reason | JSDoc | unit with a fixture IPC: stale record + failing observers → nothing green |
| main-process IPC | ADR 0041 §2.2 (one writer, the host-effect module); D#18965 C rejected | six channels, JSON in/out, no secret value crosses | a missing channel → the step's action is an operator instruction | `harness/main.mjs` JSDoc | unit: the preload/IPC contract spec |
| the credential step | `attachPlane()` precedent (`PlaneSetupPanel`) | main's window only; the card renders text | encryption unavailable → the shipped refusal line | JSDoc | e2e: no credential string in the renderer's DOM or state |
| the progress line | #12 | bound to the step Store; `n of m` + the current reason | — | JSDoc | unit: bindings update with the store |
| density record | #351 point 7 | decisions + manual actions counted per path | — | epic comment | e2e receipt on the fixture plane |

## Acceptance Criteria

- [ ] AC-1 The OQ2 witness: boot the vessel with no Brain and no config → the Create card is the primary content; dismiss it before any step ran → the switcher is live, Connect is reachable, every empty pane is labelled with what will appear there. E2E (packaged smoke or the served cockpit with a fixture IPC).
- [ ] AC-2 Projection: with a record holding `accepted` receipts and a fixture IPC whose observers fail, no step renders green; with observers green and an empty record, observation steps render `ok`. Unit.
- [ ] AC-3 The three questions: placement renders both budgets from #685's JSON and disables local presets on `pressure: 'swapping'` with the reason; preset cards render dimension, workload and floor from #686's table; the PAT step opens main's window and the renderer never holds the value (DOM + provider state asserted). Unit + e2e.
- [ ] AC-4 Resume through the other renderer: after the CLI accepted an effect, the card shows it as accepted history and offers no replay; an interrupted effect shows `reconcile-required` until a fresh observation settles it (ADR 0041 §3, cockpit side). Unit with the record fixture.
- [ ] AC-5 The progress line follows the step Store; first persistence fires the quiet confirmation once (the #14 TTFP event) and the card retires from the primary slot. E2E on the fixture plane.
- [ ] AC-6 Density: the completed run's decisions and manual actions are counted and recorded on #351. E2E receipt.
- [ ] AC-7 Visual: the card at the cockpit's token system (#13), goldens re-captured from a full visual run; the design seat's capture review attached before the PR leaves draft.
- [ ] AC-8 *(post-merge)* the first outside host's run recorded on #351 with its density count.

## Out of Scope

Host effects, the record and the CLI (neomjs/neo-agent-brain#679); the probe (#685) and presets (#686) themselves; the `*File` credential adapter; Home's doors (#244 / #341 / #342) beyond linking to them; guides (neomjs/neo-agent-brain#86); the Accounts add-agent form (#245).

## Avoided Traps

A modal wizard (the shell spec's named anti-pattern). A second host-effect implementation in the renderer or in main (one module, imported). Browser storage or a provider field as the status store (the `● streaming` anti-anchor; #181's target-binding lesson: a target switch retires everything shown). A hand-mapped step array instead of a Store. A credential value over IPC. A green step derived from a receipt.

## Related

#351 (parent) · #12 (the rule) · #7 (the vessel) · #13 (design conformance) · #14 (TTFP) · #342 (Connect door) · #244 / #341 (Home) · #181 (target binding) · #24 (the cockpit view layer conforms to the component library — the same idiom gate) · neomjs/neo-agent-brain#679 · neomjs/neo-agent-brain#685 · neomjs/neo-agent-brain#686 · ADR 0041

Decision Record impact: aligned-with ADR 0041 (the cockpit projects, never writes); aligned-with the #12 shell specification.

unowned-rationale: the design of this card is the design seat's (author) — a capture or mock lands on this ticket before a builder starts; the build waits for neomjs/neo-agent-brain#679's step JSON. First refusal when that lands: @neo-fable (#12's author, #383's cockpit lift), then @neo-opus-grace / @neo-opus-ada. Behind the Claude Desktop seat path by the operator's order of 2026-10-01.

Sweeps: live latest-open sweep — the latest 20 open Institution issues read at 2026-10-01T13:36:51Z (newest #382), no equivalent; A2A in-flight sweep — the last messages at 13:37Z, all read-states, no `[lane-claim]` / `[lane-intent]` on a setup card (claims today: #382/#383 drop zones, #687 Codex trust, #684 GitLab seat, #681/#683 seat subs); Memory Core rationale sweep — the OQ2 trail lives on D#18965 (`DC_kwDODSospM4BG0VZ`) and #12, both re-read today; own-assignment sweep — #374, #351, #42, #24, #19, #17, #16, #15 open to me, #24 adjacent (same idiom gate), none a setup card; structure map — Brain-hosted, N/A here; owning folders cited: `apps/agentos/view/PlaneSetupPanel.mjs` (sibling precedent) and `harness/main.mjs` (the IPC owner).

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "cockpit setup card create door projector recipe step store IPC main process no gate dismissible"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481

## Timeline

- 2026-10-01T13:38:30Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T13:38:30Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T13:38:30Z @neo-fable-clio added the `ai` label
- 2026-10-01T13:38:30Z @neo-fable-clio added the `architecture` label
- 2026-10-01T13:38:30Z @neo-fable-clio added the `design` label
- 2026-10-01T13:38:34Z @neo-fable-clio added parent issue #351
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T15:16:05Z @neo-fable-clio cross-referenced by #697
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351

