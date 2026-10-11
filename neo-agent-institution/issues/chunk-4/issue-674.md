---
id: 674
title: 'An installed update keeps running seats on their generation: versioned runtime roots'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
assignees: []
createdAt: '2026-10-10T23:03:34Z'
updatedAt: '2026-10-10T23:03:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/674'
author: neo-fable-clio
commentsCount: 0
parentIssue: 7
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
# An installed update keeps running seats on their generation: versioned runtime roots

## Context

Three Fleet Manager updates in four days did not install: Emmy's drain on 10-07 (Ada's two old-bundle MCP children had to be SIGTERMed by consent), Emmy's cut on 10-09 (retired after the operator restored the sessions — detached harness survival is the intended contract, #652 / #653), and Sophie's tonight (Institution #12, [6103009238](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-6103009238): the installer's bundle-use guard found 70 MCP-side processes still executing from the installed bundle — 26 `fleetMcpLauncher.mjs`, 6 `stdioToStreamableHttp.mjs`, 38 `mcp-server.mjs` — and swapped nothing; the smoke-tested candidate cb82cb2c / 98e52e9e / a852b8ca is preserved). The operator's goal is on record (2026-10-10 12:48Z): *an FM update should not pause sessions.* Today the only supported path (#259, corrected by #653) is: stop every seat, census to zero, install `--quit --open`, start the fleet — a fleet-wide stop for every cut.

**Design authority:** the operator's stated goal (12:48Z) over the current guidance of #259 / #653, which documents the limitation, not a decision to keep it. **Decision Record impact:** `amends` ADR 0034 §2.5 (distribution: the whole-package update stays; a runtime-generation clause is added — see The Fix). The amendment is the first PR.

## The Problem

Every seat's stdio MCP runtime executes **from the installed bundle**: the Fleet generates each seat's MCP command as the FM's own Electron binary (`process.execPath`, run as Node) plus `ai/mcp/client/fleetMcpLauncher.mjs` under the bundle's `organism` root (neomjs/neo-agent-brain `ai/services/fleet/prepareManagedAgentWorkspace.mjs:42, :317, :508`); the launcher resolves `INSTALL_ROOT` from its own location (`fleetMcpLauncher.mjs:29`) and runs the servers with the same executable (`:199–208`). Detached harnesses survive an FM quit by design, so their MCP processes keep the bundle's executable open, and the installer's guard (`harness/install.mjs:13–15, :70–71, :121–124`) rightly refuses to replace a bundle in use. The guard is correct; the dependency is the defect.

## The Architectural Reality

- `harness/install.mjs` (#473, Vega): replaces `Neo Harness.app` in place, keeps one rollback, census of every executable inside any `Neo Harness*.app`, `--quit` asks only the canonical FM to quit.
- Fleet custody already separates data from the replaceable bundle: `<userData>/brain/fleet` holds registrations, credentials and keys (#345 / #346). The **runtime** is the last thing still inside the bundle that running seats depend on.
- ADR 0034 §2.5: two-speed updates — the shell rarely, the organism by shipping a new whole package; "no self-mutating installed app in v1"; partial in-place organism updates are the recorded alternative, §2.5.3 the revisit point. A versioned runtime root is **not** a partial in-place update: the package stays whole and immutable; what changes is that running seats execute from a generation copy of the package's runtime, so the bundle swap no longer waits on them.
- The Brain half (filed beside this: the seat MCP command and the launch-admission answer carry the generation root) is neomjs/neo-agent-brain's leaf; this leaf is the installer, the shell and the generation lifecycle.

## The Fix

1. **A generation root per installed package:** on install (and on first run of a package that has none) the shell materialises the package's `organism` runtime under `<userData>/brain/runtime/<product-revision>/` — a copy, immutable, with the Node-capable executable it needs (the bundle's Electron copied beside it, or a pinned Node) — and records it in a generations ledger (`revision · path · installedAt · seatsUsing`).
2. **Seats launch from the generation root**, never from the bundle: the Fleet's seat MCP command points at `<generationRoot>/ai/mcp/client/fleetMcpLauncher.mjs` with the generation's executable (the Brain leaf); a running seat keeps its generation until its next Start.
3. **The installer's guard counts bundle users only:** with seats on generation roots the census of the bundle is the FM itself; `--quit --open` swaps without stopping a seat. The guard's refusal stays for any remaining bundle user (an old seat not yet migrated) and names it.
4. **Generation retirement:** a generation is removed when its `seatsUsing` census is zero and it is not the installed package's; the FM shows per seat which generation it runs and offers *Restart on the installed generation* — the operator's hand, per seat or per fleet, never automatic.
5. **The ADR amendment first:** ADR 0034 §2.5 gains the runtime-generation clause (whole package, generation roots, retirement by census); #259's guidance is rewritten from "stop every seat" to "seats keep their generation; restart them when you choose".

## Acceptance Criteria

- [ ] AC-1 — ADR 0034 §2.5 carries the runtime-generation clause (merged before the shell change); #259's update guidance is rewritten accordingly.
- [ ] AC-2 — Installing a package materialises `<userData>/brain/runtime/<revision>/` with the runtime and its executable and records the generation; a second install of the same revision is a no-op (unit on the shell's install path, filesystem fixture).
- [ ] AC-3 — With every seat on a generation root, `harness/install.mjs --quit --open` replaces the bundle while the seats keep running; the census of the bundle reports only the FM's own processes (unit with a fixture census; installed witness in AC-6).
- [ ] AC-4 — A seat started after the install runs from the installed generation; a seat started before it keeps its generation until restarted; the FM shows each seat's generation (unit + visual baseline).
- [ ] AC-5 — A generation with zero `seatsUsing` that is not the installed one is retired by the FM's maintenance, never while a seat uses it (unit).
- [ ] AC-6 *(installed, post-merge)* — on Institution #12: eight seats running, the next candidate installed without stopping any seat, every seat's MCP tools still answer, each seat restarted on the new generation by the operator's hand at his pace; recorded with the census before and after.

## Out of Scope

Auto-update feeds and signing (ADR 0034 §2.5 E7); the Brain's launcher and admission changes (the Brain leaf beside this); killing or stopping any peer process (never; the guard stays).

## Avoided Traps

- **Loosening the guard** — it is right; the dependency is wrong.
- **A partial in-place organism update** — ADR 0034 §2.5 rejected it; the package stays whole.
- **Stopping seats from the installer** — the operator restored sessions on 10-09 for exactly this; detached survival is the contract.
- **Automatic migration of running seats** — a restart is the operator's hand.

## Related

#12 (tonight's refusal, Sophie's receipt) · #7 (the shell epic — parent) · #473 (the installer) · #259 / #652 / #653 (the current guidance) · #345 / #346 (custody outside the bundle) · #424 (row 5: ordinary supported recovery — the update is one of its named failures) · ADR 0034 §2.5 · the Brain leaf filed beside this (seat MCP commands from the generation root) · neomjs/neo-agent-brain#964 (launch admission under load — the launcher's other defect this week).

## Sweeps (ticket-create §1)

Live latest-open sweep: the latest 30 open issues of this repository at 2026-10-10 23:02:07Z (newest #666), filtered for update / install / bundle / runtime: none equivalent (#17 harness demotion is adjacent, different). A2A in-flight claim sweep: Sophie's Task to me (22:58Z, `institution-12-mcp-window`, Blocked — asks for the owner and a window or a runtime separation; she retains the artifact); no claim on the fix. Memory Core rationale sweep: Vega 2026-10-09 22:44Z (the 141-process census; "a revision-scoped runtime for seats is a post-v1 design question") and 23:05Z (detached survival intended; #653); Ada 2026-10-07 18:21Z (her two old-bundle MCP children); my 2026-10-10 12:58Z DM to Sophie (blue/green versioned roots + per-generation root in the admission answer = ADR-class). Reflective pause (friction-driven): the root cause carried is the bundle dependency, not the guard. Own-assignment sweep: #507, #505, #351 — none. Placement (§1c): `harness/install.mjs` and the shell's install path; the Brain structure map for `ai/services/fleet` was run tonight for the sibling leaf. Design authority: the operator's 12:48Z goal.

**unowned-rationale:** the ADR amendment is the design seat's first PR (Clio drafts it); the shell and installer change is for the installer's and shell's authors to self-select (Vega wrote #473, Emmy #259 / #653, Ada holds row 5) — after the v13.2 cut.

Origin Session ID: 7885601f-b39c-4b4b-b246-f768b2157a7c
Retrieval Hint: "versioned runtime roots generation install guard bundle users seats keep generation FM update without stopping seats"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 7885601f-b39c-4b4b-b246-f768b2157a7c

## Timeline

- 2026-10-10T23:03:35Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T23:03:36Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T23:03:36Z @neo-fable-clio added the `ai` label
- 2026-10-10T23:03:36Z @neo-fable-clio added the `architecture` label
- 2026-10-10T23:04:27Z @neo-fable-clio added parent issue #7
- 2026-10-10T23:04:42Z @neo-fable-clio cross-referenced by #12

