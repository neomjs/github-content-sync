---
id: 1144
title: Neural Link fails after an FM update unless the Bridge is keyed by hand
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-18T19:16:38Z'
updatedAt: '2026-10-18T20:10:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1144'
author: neo-opus-vega
commentsCount: 2
parentIssue: 732
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
# Neural Link fails after an FM update unless the Bridge is keyed by hand

Part of neomjs/neo-agent-institution#732 (an FM update never requires closing a harness, and the operator never types a command to get there).

## Context

After the 2026-10-18 FM install (Brain `9cd1528b`, which carries the #1141 fix), every FM-started seat's Neural Link row was still refused. The reason changed from `credential-missing, bridge-token` (#1140) to `Neo MCP launch refused (credential-unproven, bridge-token)`: 8 times between 19:10 and 19:13Z in one seat's Claude Desktop launcher log, and on every seat after Start fleet.

**Observed, read-only:**
- Port 8081's Bridge is `run-bridge.mjs`. The host-edge orchestrator (`com.neomjs.agent-os-host-edge`) runs it from a separate Brain checkout at `feb29653` (#1044), 44 commits behind `dev`.
- That checkout's Bridge has no verifier role: `bridge_verifier` appears in neither `Bridge.mjs` nor `BridgeProtocol.mjs`.
- The Bridge process has no `NEO_FLEET_BRIDGE_PUBLIC_KEY`. Neither do the launch agent's environment nor its env file.

## The Problem

Since #980 / #1022 (`e41b8a96`), a packaged FM proves each seat's bridge token by probing the Bridge's verifier role (`probeBridgeVerifier`). By design it refuses a Bridge that is old (no verifier), legacy (no key) or holds another key. Nothing in the product provisions that key, or checks the Bridge before an update:
- The Bridge reads only `process.env.NEO_FLEET_BRIDGE_PUBLIC_KEY` (`ai/mcp/server/neural-link/Bridge.mjs:157`), and no Brain or Institution code writes it.
- The NL server's own Bridge spawn (`ai/services/neural-link/ConnectionService.mjs:989`) passes no key either.
- The required order (refresh the host-edge Bridge before the FM bundle) exists only in #1022's post-merge note, not as an installer step or check.

So an FM update silently breaks Neural Link for every seat. An outside operator has neither a Bridge refresh procedure nor a message pointing at one.

Design authority: the operator, 2026-10-18 (quoted in #732), and #1022's own deployment order, which nothing enforces.

## The Architectural Reality

- **Key source:** `FleetRegistryService.getBridgePublicKey()` (SPKI PEM of the registry's Ed25519 signing key, non-secret), documented as "provisioned to the Bridge at startup via `NEO_FLEET_BRIDGE_PUBLIC_KEY`". Who provisions it is unspecified.
- **Proof:** `McpLaunchAdmissionService` → bridge-token owner `prove` → `probeBridgeVerifier({port, publicKeySha256})`: proved / refused / unanswered.
- **The Bridge on this machine** is owned by the host-edge orchestrator (`ai/daemons/orchestrator/hostEdge.mjs`), deployed apart from the FM bundle. In a packaged FM without that orchestrator, the first NL server spawns one, without a key.

## The Fix

The Fleet owns its Bridge's verify key, end to end:

1. **Provision.** Every Bridge the packaged Fleet relies on receives the registry's public key through a product path, never by hand. Candidates, a fork for the #980 owner: the Fleet starts and owns the Bridge it proves against; or the Fleet publishes the public key to a non-secret, well-known location that any Bridge reads at start.
2. **Detect.** At boot the Fleet probes the Bridge's verifier once. An old, unkeyed or foreign Bridge becomes a named roster state with its next step, rather than one opaque `credential-unproven` per launch. The words need a design read before the PR.

## Acceptance Criteria

- [ ] AC-1: A Bridge started on the product path receives the registry's public key without operator-provided env, and a seat's bridge-token proof against it returns `proved`. Unit, red at `9cd1528b`.
- [ ] AC-2: With an old or unkeyed Bridge on the NL port, Fleet boot reports one named state with its next step. Unit; the words carry a design read on this ticket.

## Out of Scope

- Refreshing this machine's host-edge checkout today: an operational step, done separately.
- The handoff and installer leaves under #732.

## Related

neomjs/neo-agent-institution#732, #980, #1022, #1140 / #1141, #1142, #1143.

Live latest-open sweep: latest open Brain issues at 2026-10-18T19:18Z (#1143, #1142 newest); no equivalent. Org search ("bridge verifier", "BRIDGE_PUBLIC_KEY", "credential-unproven bridge-token", "host-edge Bridge"): only closed #980 and neomjs/neo#13172. Memory Core sweep: Sophie's 10-14 note recording the manual deployment order (#1022); no product owner found.

Origin Session ID: c092b691-0e97-4623-b91a-86a8c9ca2e84

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-18T19:16:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T19:16:40Z @neo-opus-vega added the `bug` label
- 2026-10-18T19:16:40Z @neo-opus-vega added the `ai` label
- 2026-10-18T19:16:40Z @neo-opus-vega added the `agent-os` label
- 2026-10-18T19:16:42Z @neo-opus-vega added parent issue #732
### @neo-opus-vega - 2026-10-18T19:26:36Z

Operational fix applied on the team's machine (operator go, 19:21Z). This is the manual provisioning this ticket exists to retire:

1. Stopped the host-edge agent (`launchctl bootout`); the plist was backed up first.
2. Moved `/Users/Shared/agent-os/neo-agent-brain` from `feb29653` to `9cd1528b`. The checkout was clean and the `package.json` delta is scripts and exports only.
3. Derived the public half of the FM registry's `signing.key` without creating anything. The fingerprint `f94cd103…` matches the seats' recorded `bridgeAuthority`. Set it as `NEO_FLEET_BRIDGE_PUBLIC_KEY` in the plist env.
4. `launchctl bootstrap`. A new Bridge on 8081 carries the key.

**Verification:**
- `probeBridgeVerifier({port: 8081, publicKeySha256: f94cd103…})` → `{"verdict":"proved"}`.
- One seat's own Neural Link launch row, run with only its row env, was admitted. The server answered `initialize` (`neo-neural-link 1.0.0`) and `tools/list` (60 tools). `healthcheck` returned `healthy` with `bridge.connected: true` 5 s after start.

Remaining: Claude Desktop seats stopped retrying their NL row after the earlier refusals, so each running seat needs an NL reconnect (seat restart or connector toggle).

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-18T19:31:19Z @neo-opus-vega cross-referenced by #1145
### @neo-opus-vega - 2026-10-18T20:10:26Z

**Handover (Vega, session sunset 2026-10-18).** The operational fix is in place on the team's machine ([6254143122](https://github.com/neomjs/neo-agent-brain/issues/1144#issuecomment-6254143122)): host-edge at `9cd1528b`, and `NEO_FLEET_BRIDGE_PUBLIC_KEY` (fingerprint `f94cd103…`) in the launch agent's env. The plist backup is `com.neomjs.agent-os-host-edge.plist.before-vega-1144-20261018`.

The product fix has not started. It is a design fork for the #980 owner: a Fleet-owned Bridge, or a published non-secret key file the Bridge reads.

Until it lands, any host-edge refresh must keep that env key, or every seat's Neural Link is refused again.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-18T20:10:29Z @neo-opus-vega cross-referenced by #1140
- 2026-10-18T20:11:02Z @neo-opus-vega cross-referenced by #732

