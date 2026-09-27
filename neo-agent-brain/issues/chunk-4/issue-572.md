---
id: 572
title: Fleet derives a seat's clone and harness home under one agents root
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T10:30:22Z'
updatedAt: '2026-09-27T10:30:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/572'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
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
# Fleet derives a seat's clone and harness home under one agents root

## Context

Sub of #571. Fleet provisions a seat into two trees beneath the plane's data root, under hashed names:
- **Clones:** `deriveAgentRepoPath` → `<managedRoot>/<id>-<sha12>/<owner-repo>-<sha12>`. `managedRoot` is `path.join(AiConfig.fleet.dataDir, 'repos')`, computed separately in `ai/services/fleet/devFleetServer.mjs:97` and `fleetServer.mjs:1284`.
- **Harness home:** `deriveAgentInstanceHome` → `<fleet.instanceRoot>/<id>-<sha12>/<type>-<sha12>`. `fleet.instanceRoot` is a plane-member leaf defaulting to `<planeDataRoot>/fleet/instances` (`ai/configBase.mjs:312`).

So a seat Fleet adopts starts in a fresh clone: a fresh cwd, which means empty Claude file-memory. And the layout #571 settles on (`/Users/Shared/agents/<id>/<owner>/<repo>`) can never be Fleet's.

## The Problem

- **Two roots, where the layout needs one folder per agent.**
- **The hashes exist only because sanitizing is lossy.** Refusing an invalid value instead of rewriting it makes them unnecessary, and the path becomes readable.
- **Both roots resolve beneath the plane**, but a seat's working trees and path-keyed memory must outlive any plane. ADR 0019 §10.9: what has to survive the plane must not resolve beneath it.

## The Architectural Reality

- **Derivations:** `deriveAgentRepoPath.mjs` and `deriveAgentInstanceHome.mjs` are pure and contained, and keyed by agent id, never `githubUsername`.
- **Managed root:** `FleetManager.getManagedRoot()` throws unless an entrypoint injects it.
- **Instance root:** `FleetLifecycleService.getInstanceRoot()` returns `this.instanceRoot || AiConfig.fleet.instanceRoot` (`:1280`). `startAgentProvisioned` and `prepareManagedAgentWorkspace` take the root as a parameter.
- **Readers of `fleet.instanceRoot` / `NEO_FLEET_INSTANCE_ROOT`:** `PLANE_MEMBER_PATHS` (`configBase.mjs:2731`), `ai/scripts/lint/config-leaf-parity.json:235`, `deploy/cloud/docker-compose.dev.yml:63`, `ai/scripts/diagnostics/captureParityLatencyPair.mjs:1014`. Across repos: the Institution's `harness/brain.mjs` (`resolveBrainPaths`, `buildBrainProfile`, `buildPackagedBrainEnv`), which adapts at its next Brain pin.
- **Nothing to migrate:** no provisioned seat exists under the hashed layout on this machine (no `brain/fleet` under any app's userData).

## The Fix

1. **One leaf:** `fleet.agentsRoot` (`NEO_FLEET_AGENTS_ROOT`), `planeMember: false` with its reason. It takes the `backupPath` shape: `leaf(path.resolve(os.homedir(), '.neo-ai', 'agents'), …)`, a default that derives from neither the plane nor a checkout. This machine sets `/Users/Shared/agents`.
2. **Plain paths:** `deriveAgentRepoPath` → `<agentsRoot>/<id>/<owner>/<repo>`; `deriveAgentInstanceHome` → `<agentsRoot>/<id>/harness/<type>`. Each segment is validated, not hashed:
   - a value outside the safe charset is refused, never rewritten;
   - lowercase only, because the default macOS volume is case-insensitive and `Ada` and `ada` would share a folder;
   - the owner `harness` is refused, so a clone can never land in a harness home.
3. **One reader path:** the entrypoints inject `AiConfig.fleet.agentsRoot` instead of the `repos` join, deleting both derivations, and `getInstanceRoot()` reads the same leaf. `fleet.instanceRoot` and `NEO_FLEET_INSTANCE_ROOT` retire with their readers: the parity JSON, the member list, the dev compose and the parity diagnostic.
4. **Tests keep injecting a tmp root.** No spec provisions into the leaf's default.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.agentsRoot` / `NEO_FLEET_AGENTS_ROOT` (new) | ADR 0019 §2, §10.9 | host root for every seat's clones and harness home; `planeMember: false` with a reason | `~/.neo-ai/agents` | leaf JSDoc, `learn/agentos/OwnAgentTeam.md` | config arm: resolves outside `plane.dataRoot`; the member walk excludes it |
| `fleet.instanceRoot` / `NEO_FLEET_INSTANCE_ROOT` (retired) | this ticket | removed with every reader | none (env-var rename rule) | `config-leaf-parity.json` | census arm: no reader left |
| `deriveAgentRepoPath` | this ticket | `<root>/<id>/<owner>/<repo>`, validated | refuses | JSDoc | unit arms, refusals included |
| `deriveAgentInstanceHome` | this ticket | `<root>/<id>/harness/<type>`, validated | refuses | JSDoc | unit arms |
| FleetManager managed root | this ticket | entrypoints inject the leaf | throws when uninjected (unchanged) | JSDoc | server arms |

## Decision Record impact

aligned-with ADR 0019: a declared leaf with §10.9's non-member placement. The retired member updates `PLANE_MEMBER_PATHS` and the §10.5 member-completeness spec.

## Acceptance Criteria

- [ ] AC-1: `deriveAgentRepoPath({agentsRoot, agentId: 'neo-fable-clio', repoSlug: 'neomjs/neo'})` returns `<agentsRoot>/neo-fable-clio/neomjs/neo`, and `deriveAgentInstanceHome` returns `<agentsRoot>/neo-fable-clio/harness/claude-desktop` (unit).
- [ ] AC-2: uppercase, `..`, separators, the owner `harness` and any segment outside the charset are refused with a named error; nothing is sanitized (unit).
- [ ] AC-3: `fleet.agentsRoot` resolves outside `plane.dataRoot` by default and carries `planeMember: false` with a reason; the member-completeness spec stays green without it (unit).
- [ ] AC-4: neither `fleet.instanceRoot` / `NEO_FLEET_INSTANCE_ROOT` nor either `repos` join has a reader left in the Brain (census arm); the dev compose and the parity diagnostic follow.
- [ ] AC-5: a provisioned start derives both paths from the one leaf (`FleetLifecycleService` and `FleetManager` arms with an injected tmp root).

## Out of Scope

- Moving existing seats (the migration leaf under #571).
- The Institution's env in `harness/brain.mjs`, which follows at its next Brain pin.
- The zshenv arm, the LaunchAgents and per-clone `.env` provisioning.
- Seats on other machines.

## Related

Parent #571 · #335 · #142 (reconciling memory when a seat is removed) · neomjs/neo-agent-institution#280 · ADR 0019

Sweeps (10:27Z):
- Live latest-open (20): no equivalent.
- A2A lane-claims (last 30): none on seat paths.
- Memory Core: my own plan (08:37Z) and Clio's 2026-08-28 env move. Neither touches Fleet's derivation.
- Own assignments: #142 covers removal only.
- Structure map (`4bc885b`): `ai/services/fleet/` owns both derivations; no new file.

Origin Session ID: f3d50317-fe3b-4773-b4ac-db05e1fa6812
Retrieval Hint: `query_raw_memories("Fleet deriveAgentRepoPath deriveAgentInstanceHome agentsRoot plain segments validated not hashed")`

## Timeline

- 2026-09-27T10:30:22Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T10:30:23Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T10:30:23Z @neo-opus-ada added the `ai` label
- 2026-09-27T10:30:23Z @neo-opus-ada added the `architecture` label
- 2026-09-27T10:30:23Z @neo-opus-ada added the `agent-os` label
- 2026-09-27T10:30:28Z @neo-opus-ada added parent issue #571
- 2026-09-27T11:04:13Z @neo-opus-ada referenced in commit `77e9748` - "feat(fleet): a seat's clones and harness homes derive under one agents root, in readable paths (#572)

`fleet.agentsRoot` (`NEO_FLEET_AGENTS_ROOT`) is the one root of every agent's folder: clones at
`<id>/<owner>/<repo>`, harness homes at `<id>/harness/<type>`. It is not a plane member — a seat's
working trees and path-keyed memory must outlive any plane — and defaults to `~/.neo-ai/agents`,
the `backupPath` shape.

The derivations validate instead of hashing: a segment outside lowercase letters, digits, `.`,
`_` and `-` is refused, never rewritten, so each segment is the raw value and distinct values
cannot collide. Lowercase only, because the default macOS volume is case-insensitive; the owner
`harness` is reserved so a clone never lands in a harness home. One shared rule replaces the two
byte-identical hash helpers.

`fleet.instanceRoot` / `NEO_FLEET_INSTANCE_ROOT` retire with every reader, and both entrypoints
inject the leaf instead of each deriving `path.join(fleet.dataDir, 'repos')`. The derivation
specs and `planeConfig.spec` join brain-unit's run list; comments in the touched files describe
behavior instead of citing tickets."
- 2026-09-27T11:04:13Z @neo-opus-ada referenced in commit `d7efd0c` - "test(fleet): a provisioned start reads both roots from the one agents-root leaf (#572)"
- 2026-09-27T11:04:26Z @neo-opus-ada cross-referenced by PR #573
- 2026-09-27T11:38:15Z @neo-opus-ada referenced in commit `889bd00` - "test(fleet): the agents-root arm reads the config template, not the overlay (#572)

Tests resolve the committed template; the Playwright resolver maps the
service's own config import onto it, so the arm reads the same Provider."

