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
updatedAt: '2026-10-18T23:07:10Z'
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

On 2026-10-18 three P0s reached the installed FM: #1136 (a Codex seat refuses its own MCP rows after the runtime move), #1138 (a seat gone at the next boot recovers as `exited`) and #1140 (a fresh Start's Neural Link launch is refused, because the Bridge-token owner gets a null registry). All three were Brain defects in the joins between Fleet services. The unit specs stub exactly those joins: #1158 was a leaked `launchAdmission` stub in `FleetLifecycleService.spec`. The integration tier covers Memory Core and the Knowledge Base only (`test/playwright/integration/`). No test runs a seat's lifecycle through real collaborators.

## The Problem

A Fleet change can pass every unit spec and still break Start, restart or recovery. Today that shows only on the installed FM, after an install that then costs a corrective one. That is the trap neomjs/neo-agent-institution#732 exists for.

## The Architectural Reality

Every input a hermetic Start needs already has a seam:
- **Harness binary.** `fleet.harnessBinaries.*` is an AiConfig leaf with an env binding: `NEO_FLEET_CLAUDE_DESKTOP_BIN`, `NEO_FLEET_CODEX_DESKTOP_BIN` and so on (`FleetLifecycleService#getHarnessBinaryPath`).
- **Git identity.** A definition that declares `gitName` and `gitEmail` needs no forge read (`seatGitIdentity.mjs#resolveSeatGitIdentity`).
- **Checkout.** `ensureAgentRepo` reuses an existing valid checkout and never reclones it, so a pre-seeded local repo means no network.
- **Plane.** The integration tier already boots Memory Core and the KB servers (`test/playwright/integration/fixtures/composeWebServer.mjs`, `mcpClient.mjs`).
- **Verbs.** Start, restart and definition go over the fleet server's own wire (`FleetControlBridge`: `defineAgent`, `setRepo`, `startAgent`, `restartAgent`), which is the path the FM's cockpit takes.

The family configs are `<instanceHome>/claude_desktop_config.json` for Claude Desktop, and `<instanceHome>/codex-home/config.toml` plus the repo's `.codex/config.toml` for Codex Desktop (`prepareManagedAgentWorkspace.mjs`). Each MCP row spawns `fleetMcpLauncher.mjs`, which redeems launch admission before its server starts.

## The Fix

1. **The journey, `ai/services/fleet/seatJourney.mjs`** (a sibling of `startAgentProvisioned.mjs`). It drives a fleet server process from outside, through its wire, given that server's endpoint, its bearer and a restart callback:
   - define stand-in seats;
   - Start, Stop, Start;
   - stop the fleet server uncleanly while the seat's process is gone, restart it on the same root, and Start again.

   An optional runtime-move leg starts the seats under one runtime generation and starts them again under a second one. Each step returns a structured row: the family, the step, pass or fail, the reason, and config digests. It ships in `ai/`, so the packaged rehearsal (neomjs/neo-agent-institution#736) runs the same journey against a candidate's organism.
2. **The stand-in, beside `ai/scripts/fleet/onboardPeer.mjs`** (structural pre-flight at implementation). It is an executable a family's harness binary points at. It reads the family's config where the real app reads it, spawns every MCP row the way the app would, runs `initialize`, `tools/list` and one `healthcheck` per row, records the results in its instance home, and stays alive until SIGTERM. It parses JSON and TOML itself and never imports the writer it checks. Codex keeps the real `codex` CLI as its config consumer when that CLI is installed; when it is not, the family is skipped and the journey says so.
3. **`test/playwright/integration/ai/fleet/SeatJourney.integration.spec.mjs`** runs the journey in the integration job, against a fleet server started from the source tree, with its own Bridge on an allocated port.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `seatJourney.mjs` (new) | this ticket | Drives one fleet server through its wire; returns step rows | — | module JSDoc | AC-1–4 |
| Stand-in executable (new) | this ticket; the families' config shapes | Answers as a harness: reads its config, spawns its rows, records results | A family without its consumer is skipped, with the reason | JSDoc | AC-1, AC-4 |
| `fleet.harnessBinaries.*` (existing) | AiConfig leaves (ADR-0019) | Set through the env binding of the fleet server child only | — | — | AC-1 |

ADR-0019 has been read for this ticket. The journey sets AiConfig leaves only through the child's env bindings: no runtime mutation, no re-derivation and no pass-along.

## Decision Record impact

Aligned with #1051, the continuity contract: the journey exercises it and does not change it. ADR-0019 is aligned as stated above.

## Acceptance Criteria

- [ ] AC-1: On `dev`, the journey passes for Claude Desktop and Codex Desktop. Every MCP row answers on every Start, Neural Link's `healthcheck` reports `healthy` with `bridge.connected: true`, and the recovered seat reads `stopped`.
- [ ] AC-2 (red first): with the fixes reverted, the journey fails at the right step. That is a checkout at `d6a905dc`, which predates #1137, #1139 and #1141 (`d6a905dc..9cd1528b` holds exactly those three). The verdict names `exited` at recovery (#1138) and the NL launch refusal (#1140). With the runtime-move leg, it also names the Codex rows' refusal (#1136).
- [ ] AC-3: After the second Start, each family's config bytes equal those after the first, including tables the Fleet does not own. A seeded rewrite fails the step (unit test).
- [ ] AC-4: The run makes no request beyond loopback, and teardown leaves no process in the run's process group.
- [ ] AC-5: The integration job stays inside its 30-minute budget. The PR records the journey's wall time.

## Out of Scope

- Running the journey against a packaged candidate on the operator's machine: neomjs/neo-agent-institution#736.
- Stand-ins for Antigravity, Kimi Code, OpenCode, Claude Code and the Codex CLI: one row per family, later.
- Proving that a real harness app accepts the config. The installed walk keeps that residual (neomjs/neo-agent-institution#12).

## Avoided Traps

- **More stubs at the joins.** The defects lived there, and so did #1158's leak.
- **Real harness apps in CI.** They open windows and need accounts.
- **Driving the Fleet in-process.** Recovery is a fresh boot; only a second process proves it.

## Related

#1136, #1137, #1138, #1139, #1140, #1141 · #1158 · #1051 · neomjs/neo-agent-institution#732 (epic review 6256224005) · neomjs/neo-agent-institution#736 (its consumer) · neomjs/neo-agent-institution#12

Live latest-open sweep: latest 20 open Brain issues at 2026-10-18T23:06Z; the newest is #1161. No equivalent found. Org search ("seat journey", "stand-in harness", "fleet integration test", "lifecycle end to end stand-in") found only neomjs/neo-agent-institution#736.
A2A sweep: the latest messages through 23:06Z, all read states. No claim on this scope.
Memory Core sweep: no prior seat-journey or pre-install gate design.
Structure map: `ai/services/fleet` is flat; the journey sits beside `startAgentProvisioned.mjs`, and the stand-in goes under `ai/scripts/fleet/` (structural pre-flight at implementation).

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

