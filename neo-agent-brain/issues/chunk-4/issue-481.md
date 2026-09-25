---
id: 481
title: The host edge elects the Neural Link bridge lane
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-25T11:01:05Z'
updatedAt: '2026-09-25T11:35:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/481'
author: neo-opus-vega
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
closedAt: '2026-09-25T11:35:02Z'
---
# The host edge elects the Neural Link bridge lane

## Context

Operator goal, 2026-09-25: the Fleet Manager's Electron shell on the team machine, every peer entering the same instance via Neural Link. Measured on this machine today (receipts on neomjs/neo#18965, comments 18597857 and the 11:00Z correction):

- Three bridges run: 18081 (Ada's seat, spawned from a Brain worktree), 8093 (Mnemo's, from a Brain checkout) and 8081, an orphan from a 2026-09-24 Playwright run of Euclid's (identity `@neo-gpt`, log under a `$TMPDIR` worker dir, parent reparented to launchd). None is supervised.
- Every seat's Neural Link MCP server is launched with `--cwd <an engine checkout>` (the seats' desktop configs), and the engine has carried no bridge script since the split: my seat's `healthcheck` reports `cwdFinding: BRIDGE_CWD_MISSING_SCRIPT` (`ai/services/neural-link/ConnectionService.mjs:536`, the cwd's `package.json` has no `ai:server-neural-link`). So no seat spawns a bridge; each attaches to whatever listens on its port.
- The cockpit dials `Neo.config.neuralLinkUrl`, default `ws://127.0.0.1:8081` (`neomjs/neo src/ai/Client.mjs:165`); the institution app sets no override. Both cockpits that dialed today landed on the orphan.
- The orchestrator already defines the lane: `ai/daemons/orchestrator/taskDefinitions.mjs:346` `neuralLinkBridge` supervises `run-bridge.mjs` with `NEO_NL_PORT`, `singletonPort`, `duplicateListenerPolicy: 'defer'` and a TCP liveness probe, gated by `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED`. `src/composition/orchestrator/hostEdgeProfile.mjs:119` turns it off as a "host-edge-class lane this topology does not elect". The container orchestrator does not run it either (no bridge line in 24 h of its log). `run-bridge.mjs:55` binds `127.0.0.1`, so a container-hosted bridge would not be reachable from the host through a published port without a bind change.

Sweeps at 10:59Z: live latest-20 open Brain issues, no equivalent (#84, the hard-cut topology ticket, is the parent topology, not this lane); A2A last 12 messages, only my own broadcasts and DMs on the topic, no competing claim; Memory Core rationale sweep: Grace's 2026-09-22 note (sharing a bridge is the default feature, `NEO_NL_PORT` the opt-out, on the operator's "peers CAN share a bridge matters") and Euclid's 2026-07-13 map ("the orchestrator can keep it resident across agent sessions"); own-assignment sweep: none on this surface. Structure map: the closure lives in `src/composition/orchestrator/hostEdgeProfile.mjs`, the lane in `ai/daemons/orchestrator/taskDefinitions.mjs`, the closure's spec in `test/playwright/unit/ai/daemons/orchestrator/HostEdgePosture.spec.mjs`; no new file.

## The Problem

Nothing on this machine owns the bridge every seat and the cockpit depend on. Bridges appear as side effects of test runs and manual starts, die with them or outlive them as orphans, and a seat cannot recover one. The bridge the whole team is on today has a test worker's identity and log path, and stopping it would take Neural Link away from every default-port seat with no way back short of a manual `npm run ai:server-neural-link` from a Brain checkout.

## The Architectural Reality

- The host edge is the graphless launchd orchestrator that owns host-bound lanes (LM Studio supervision today). Its profile is the SSOT for the lane closure: role, state root, and every lane this role does or does not elect, stated in one place (`hostEdgeProfile.mjs` §3).
- The lane has singleton-port semantics: `duplicateListenerPolicy: 'defer'` means the host edge will not fight a foreign listener, so the orphan must be gone before the host edge takes 8081; from then on the liveness probe respawns the bridge whenever it dies.
- The port is owned by the Neural Link config provider (`ai/mcp/server/neural-link/config.mjs`, `NEO_NL_PORT`, default 8081) and read by the orchestrator entrypoint; the cockpit's default matches it, so the team needs no cockpit override.
- Security posture unchanged: the bridge binds loopback, and without a verify key it admits legacy unauthenticated agent ids (the no-FM dev posture until the Fleet Manager mints tokens, `Bridge.mjs:190`).

## The Fix

1. `hostEdgeProfile.mjs`: `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED: 'true'`, moved into the elected group with the reason (the machine's seats and cockpit share one supervised bridge); the port stays the provider's.
2. `HostEdgePosture.spec.mjs`: the closure assertion flips for this key (red-first on `dev`).
3. `ai/scripts/lifecycle/local-agent-os/README.md` topology: the Neural Link bridge is a host-edge lane on `127.0.0.1:8081`.
4. Deploy on this machine (post-merge, operator-owned): stop the orphan (PIDs 34749 → 34804), then bootout/bootstrap `com.neomjs.agent-os-host-edge` from the deploy home at the merged SHA; receipt: `neural-link-bridge.pid` under the host-edge state root, `lsof -iTCP:8081` naming a `run-bridge.mjs` child of the host edge, two seats' `healthcheck` on it.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| host-edge lane closure (`hostEdgeProfile.mjs`) | this ticket + `taskDefinitions.mjs:346` | `neuralLinkBridge` elected on the host edge | unchanged: an explicit env value still wins per key | local-agent-os README topology | spec arm + the deploy receipt |
| bridge port | `ai/mcp/server/neural-link/config.mjs` (`NEO_NL_PORT`) | unchanged (8081) | unchanged | unchanged | seats' `healthcheck.bridge.port` |

Decision Record impact: aligned-with ADR 0019 (an existing leaf flipped in the profile, read at the use site; no new leaf).

## Acceptance Criteria

- [ ] `hostEdgeProfile.mjs` elects `NEO_ORCHESTRATOR_NL_BRIDGE_ENABLED: 'true'`; `HostEdgePosture.spec.mjs` fails on `dev` for that key and passes at the head.
- [ ] The README's topology section lists the Neural Link bridge as a host-edge lane on `127.0.0.1:8081`.
- [ ] Post-merge, on this machine: with the orphan stopped and host-edge restarted at the merged SHA, `lsof -iTCP:8081` names a `run-bridge.mjs` child of the host-edge orchestrator, `neural-link-bridge.pid` exists under the host-edge state root, and two seats' `healthcheck` report `bridge.connected: true, port: 8081` on it.
- [ ] Post-merge: a kill of that bridge is followed by a respawn within the liveness cadence (receipt: two `Agent connected` epochs for one seat in the bridge log).

## Out of Scope

- The seats' `--cwd` in their desktop MCP configs (operator-owned harness configs, noted to him): with a supervised bridge a seat never needs to spawn one.
- A container-hosted bridge published on the host (needs a bind change; loopback-only is a property worth keeping).
- Signed agent tokens (FM-minted): the dev posture stays until the Fleet Manager spawns seats.

## Related

neomjs/neo#18965 (the receipts) · neomjs/neo-agent-institution#7 (the Electron shell epic; this is its "one canonical bridge" leaf) · #84 (the machine's hard-cut topology) · neomjs/neo#19065 (a per-seat port as the opt-out).

Origin Session ID: ec6c7966-ab2b-43d0-89cd-5ec2262b8424
Retrieval Hint: "host edge elects neuralLinkBridge lane orphan bridge 8081 BRIDGE_CWD_MISSING_SCRIPT"


## Timeline

- 2026-09-25T11:01:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-25T11:01:07Z @neo-opus-vega added the `enhancement` label
- 2026-09-25T11:01:07Z @neo-opus-vega added the `ai` label
- 2026-09-25T11:01:07Z @neo-opus-vega added the `agent-os` label
- 2026-09-25T11:18:18Z @neo-opus-vega referenced in commit `629dae2` - "ci(brain-unit): the two host-edge closure specs join the run list (#481)

Brain Unit collects the whole suite and executes a named list; the closure
specs that now assert the bridge election were collected, never run (#201)."
- 2026-09-25T11:20:43Z @neo-opus-vega cross-referenced by PR #483
- 2026-09-25T11:35:03Z @tobiu referenced in commit `d6c8aed` - "Merge pull request #483 from neomjs/vega/481-host-edge-elects-nl-bridge

feat(host-edge): the host edge elects the Neural Link bridge lane (#481)"
- 2026-09-25T11:35:03Z @tobiu closed this issue

