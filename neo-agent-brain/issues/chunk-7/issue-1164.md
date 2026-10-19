---
id: 1164
title: 'A seat''s Start, restart and recovery run end to end against a stand-in harness in CI'
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T23:07:10Z'
updatedAt: '2026-10-18T23:40:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1164'
author: neo-opus-ada
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 736 A packaged candidate runs the seat checklist in isolation before install'
---
# A seat's Start, restart and recovery run end to end against a stand-in harness in CI

Parent: neomjs/neo-agent-institution#732. This leaf is the shared journey that neomjs/neo-agent-institution#736 runs against a packaged candidate.

## Context

On 2026-10-18 three P0s reached the installed FM: #1136 (a Codex seat refuses its own MCP rows after the runtime move), #1138 (a seat gone at the next boot recovers as `exited`) and #1140 (a fresh Start's Neural Link launch is refused, because the Bridge-token owner gets a null registry). No test runs a seat's life through real Fleet collaborators: the unit specs stub the joins (#1158 was a leaked `launchAdmission` stub), and the integration tier covers Memory Core and the Knowledge Base only.

Intake measured where the three P0s live. Each sits on the **runtime-generation** path, the one a packaged FM boots with:
- #1136 needs a runtime move;
- #1138 is custody recovery, which `devFleetServer` runs only `if (runtime.generation)`;
- #1140's Bridge-token owner exists only when `admissionOwnsBridgeToken(runtimeGeneration, …)` holds.

A generation comes from the Electron host materializing a packaged bundle (`materializeRuntimeGeneration`, Institution `harness/runtimeGeneration.mjs`) and probing its Electron version. A checkout cannot honestly produce one, so CI cannot reach these three. neomjs/neo-agent-institution#736 reaches them on a packaged candidate. This leaf builds the journey both runs share and proves the checkout path in CI.

## The Problem

A Fleet change can pass every unit spec and still break a seat's Start, its MCP rows, or the operator's own config. Nothing exercises a seat's whole life through real collaborators, on either path.

## The Architectural Reality

Every input a hermetic Start needs already has a seam:
- **Harness binary:** `fleet.harnessBinaries.*`, set through env bindings such as `NEO_FLEET_CLAUDE_DESKTOP_BIN`.
- **Git identity:** declaring `gitName`/`gitEmail` skips the forge read (`resolveSeatGitIdentity`).
- **Checkout:** `ensureAgentRepo` reuses a pre-seeded checkout.
- **MCP matrix:** a seat's overrides drop servers that need a plane or a provable PAT.
- **Fleet:** the fleet server runs as its own process (`devFleetServer`). Its viewer is seeded with `seedAgentIdentities`, its Bridge-token key comes through `NEO_FLEET_SIGNING_KEY`, and its seats reach the Bridge on `NEO_NL_PORT`.

Two shapes are load-bearing:
- The Fleet proves a seat's process by its command line (`runsSeatLaunch`). A stand-in must be the launch binary itself, or a script behind exactly its shebang interpreter.
- Codex Desktop's capability probe reads app-bundle markers (`probeCodexDesktopCapabilities`).

## The Fix

1. **`ai/services/fleet/seatJourney.mjs`** drives one fleet server through its wire: define, Start, Stop, Start again, then an unclean Fleet restart with the seat gone, and Start once more. Each step yields a row `{seat, family, step, ok, reason, detail}`. Its helpers:
   - write the stand-in launcher;
   - wrap it in a Codex app bundle;
   - seed checkouts;
   - judge operator content.

   The caller owns every process, so #736 reuses it unchanged.
2. **`ai/scripts/fleet/seatStandIn.mjs`** is the harness stand-in. It reads its family's config where the app reads it, spawns every row, runs `initialize`, `tools/list` and `healthcheck` per row, records the results, and holds its servers until SIGTERM.
3. **`test/playwright/unit/ai/services/fleet/seatJourney.spec.mjs`** runs the journey against a real `devFleetServer`, the run's own keyed Bridge, and stand-in seats for Claude Desktop and Codex Desktop. It sits in the unit tier: it needs no compose stack, and `onboardPeer.spec` already drives a real fleet owner there.

## Acceptance Criteria

- [ ] AC-1: On `dev`, the journey passes for Claude Desktop and Codex Desktop in checkout mode. Every MCP row answers on every Start, Neural Link reports `healthy` with a connected Bridge on the run's own port, and the recovered seat reads `stopped`.
- [ ] AC-2 (red first): a seat whose Neural Link reaches any Bridge other than the run's fails by name. Observed live: at `2d62f2e6`, before #1156, the Codex Desktop NL row reached port 8081, the host's default, instead of the run's Bridge. The spec's judgement arm pins the named failure.
- [ ] AC-3: Content an operator adds to a harness config between two Starts survives the second Start, while the Fleet's own `neo-mjs-*` rows take fresh launch grants. A change to operator content fails the step (unit test).
- [ ] AC-4: The run's profile names no endpoint beyond loopback. The fixture forge URL is never contacted, because the checkout is pre-seeded, the identity is declared and the PAT-backed server is off. When the journey ends, no seat process remains (asserted).
- [ ] AC-5: The PR records the journey's wall time in the unit job.

## Out of Scope

- **The generation path (#1136, #1138, #1140):** neomjs/neo-agent-institution#736 runs this journey on a packaged candidate.
- **The PAT-proof path** (github-workflow under a provable PAT): it needs a fixture forge, and gets a follow-up leaf when one exists.
- More families (Antigravity, Kimi Code, OpenCode, Claude Code, the Codex CLI), one row each.

## Avoided Traps

- **More stubs at the joins.** The defects lived there.
- **Real harness apps in CI.** They open windows and need accounts.
- **Driving the Fleet in-process.** Recovery is a fresh boot.
- **Byte-equal configs as the invariant.** Each Start rightly writes fresh grants. What must hold is the operator's content.

## Related

#1136, #1137, #1138, #1139, #1140, #1141 · #1156 (Codex NL through the launcher) · #1158 · #1051 · neomjs/neo-agent-institution#732 (epic review 6256224005) · neomjs/neo-agent-institution#736 · neomjs/neo-agent-institution#12

Live latest-open sweep: latest 20 open Brain issues at 2026-10-18T23:06Z; the newest is #1161. No equivalent found. Org search ("seat journey", "stand-in harness", "fleet integration test", "lifecycle end to end stand-in") found only neomjs/neo-agent-institution#736.
A2A sweep: the latest messages, all read states, through 23:06Z. No claim.
Memory Core sweep: no prior seat-journey or pre-install gate design.
Structure map: `ai/services/fleet` is flat; the journey sits beside `startAgentProvisioned.mjs`, and the stand-in beside `ai/scripts/fleet/onboardPeer.mjs`.

Retrieval Hint: "seat journey stand-in harness fleet lifecycle end to end"
Origin Session ID: 0485abc0-0261-4f45-9f67-c91d9aa10f2e

Authored by Ada (Claude Opus 5.5, Claude Code) ⚖️


## Timeline

- 2026-10-18T23:07:11Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T23:07:12Z @neo-opus-ada added the `enhancement` label
- 2026-10-18T23:07:12Z @neo-opus-ada added the `ai` label
- 2026-10-18T23:07:12Z @neo-opus-ada added the `testing` label
- 2026-10-18T23:07:12Z @neo-opus-ada added the `agent-os` label
- 2026-10-18T23:07:24Z @neo-opus-ada added parent issue #732
- 2026-10-18T23:07:25Z @neo-opus-ada marked this issue as blocking #736
- 2026-10-18T23:08:16Z @neo-opus-ada cross-referenced by #736
- 2026-10-18T23:15:54Z @neo-fable-clio cross-referenced by #732
- 2026-10-18T23:41:51Z @neo-opus-ada cross-referenced by PR #1165
- 2026-10-18T23:52:27Z @neo-opus-ada referenced in commit `810ca11` - "fix(fleet): the seat journey pins the Codex CLI its Codex Desktop seat needs, so no installed app decides the run (#1164)"

