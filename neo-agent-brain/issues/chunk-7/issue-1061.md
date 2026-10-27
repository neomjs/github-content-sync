---
id: 1061
title: Preserve Homebrew and local bin PATH in Antigravity MCP invokers
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-15T23:24:19Z'
updatedAt: '2026-10-16T15:31:20Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1061'
author: neo-gemini-julia
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
closedAt: '2026-10-16T15:31:20Z'
---
# Preserve Homebrew and local bin PATH in Antigravity MCP invokers

## Context

Observed on `2026-10-15T22:55Z` after `@tobiu` restarted the managed `@neo-gemini-pro` Antigravity seat from the packaged macOS `Neo Harness.app` (PID `99813`, launched from macOS Finder/Dock). Immediately after the seat restart:

- `neo-mjs-memory-core` and `neo-mjs-knowledge-base` connected and answered `healthcheck`.
- `neo-mjs-github-workflow` (`list_issues`, `signal_state_transition`) failed every tool call with:
  `Cannot execute list_issues: GitHub CLI is not available: - GitHub CLI is not installed. Please install it from https://cli.github.com/`
- On the same host and inside the same Antigravity seat's shell, `/opt/homebrew/bin/gh` is installed and authenticated.

## The Problem

When `Neo Harness.app` is launched from macOS Finder/Dock rather than an interactive terminal, macOS supplies the minimal launchd `PATH` (`/usr/bin:/bin:/usr/sbin:/sbin`), which excludes `/opt/homebrew/bin` and `/usr/local/bin`.

1. `neomjs/neo-agent-institution/harness/brain.mjs:908-926` (`startBrainChild`) spawns `ai/services/fleet/devFleetServer.mjs` (PID `99840`) with `PATH` set to `<shims>:<node_modules/.bin>:${process.env.PATH}` (`/Applications/Neo Harness.app/Contents/Resources/organism/shims:/Applications/Neo Harness.app/Contents/Resources/organism/node_modules/.bin:/usr/bin:/bin:/usr/sbin:/sbin`).
2. When Antigravity (`Antigravity.app`) boots, its Electron main process runs a login-shell overlay so its `language_server` child has `/opt/homebrew/bin` in `PATH`.
3. However, `ai/services/fleet/generateAntigravityMcpFiles.mjs:63-66` writes each profile invoker (`<profileHome>/.neo-fleet-mcp/<key>.sh`) using `exec /usr/bin/env -i` and injects `PATH` from `processEnvironment.PATH`, which `FleetLifecycleService.snapshotAmbientProcessEnvironment()` (`ai/services/fleet/FleetLifecycleService.mjs:2804-2812`) copied verbatim from `devFleetServer.mjs`'s `process.env.PATH`.
4. Because `/usr/bin/env -i` wipes the inherited environment from Antigravity's `language_server` and replaces `PATH` with `devFleetServer.mjs`'s stripped Finder `PATH`, `ai/mcp/server/github-workflow/mcp-server.mjs` (spawned by `ai/mcp/client/fleetMcpLauncher.mjs`) cannot find `/opt/homebrew/bin/gh`.

## The Architectural Reality

- `ai/services/fleet/FleetLifecycleService.mjs:743-745` (`start` default `resolvedProcessEnvironment` for every managed seat's spawn environment) and `:3083` (`resolveAntigravityStartContext`) call `this.snapshotAmbientProcessEnvironment()` (`:3151-3163`) to capture `AMBIENT_ENV_ALLOWLIST` (`HOME`, `PATH`, `USER`, `LOGNAME`, `TMPDIR`, `LANG`, `LC_ALL`, `SHELL`, `TERM`).
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs:1490-1494` (`prepareAntigravityArtifacts`) forwards `processEnvironment` to `generateAntigravityMcpFiles({profileHome: instanceHome, rows, processEnvironment})`.
- `ai/services/fleet/generateAntigravityMcpFiles.mjs:59-67` serializes `exec /usr/bin/env -i` with `PATH=${processEnvironment.PATH}` into `<profileHome>/.neo-fleet-mcp/{memory-core,knowledge-base,github-workflow,neural-link}.sh`.
- `neomjs/neo-agent-institution/harness/brain.mjs:908-926` (`startBrainChild`) constructs `PATH` for `devFleetServer.mjs` without ensuring standard macOS user/package binary paths (`/opt/homebrew/bin`, `/usr/local/bin`) are present when `Neo Harness.app` starts from Finder/Dock.

## The Fix

1. Normalize `PATH` in `FleetLifecycleService.snapshotAmbientProcessEnvironment()` (or in a shared `PATH` helper consumed by `snapshotAmbientProcessEnvironment` and `startBrainChild`) so that on `darwin` / POSIX hosts, existing standard tool directories (`/opt/homebrew/bin`, `/usr/local/bin`) are appended to `PATH` when absent from a stripped `launchd` GUI parent's `process.env.PATH` (preserving the caller's existing segments first so shims and explicit overrides still win, and returning non-`launchd` `PATH`s unchanged).
2. Add unit test coverage in `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` and `test/playwright/unit/ai/services/fleet/generateAntigravityMcpFiles.spec.mjs` verifying that when `process.env.PATH` is Finder's minimal `/usr/bin:/bin:/usr/sbin:/sbin`, the snapshotted `processEnvironment.PATH` and generated `.neo-fleet-mcp/github-workflow.sh` preserve `/opt/homebrew/bin` and `/usr/local/bin` (after existing `PATH` entries), while non-`launchd` `PATH`s remain unchanged.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService.snapshotAmbientProcessEnvironment` (consumed by `start()`'s default `resolvedProcessEnvironment` at `FleetLifecycleService.mjs:743–745` for every managed seat's spawn env and `resolveAntigravityStartContext` at `:3083`) | `ai/services/fleet/FleetLifecycleService.mjs:3151` | Appends missing standard POSIX/macOS tool bin directories (`/opt/homebrew/bin`, `/usr/local/bin`) after existing `process.env.PATH` segments when every `PATH` segment is a `launchd` system dir (`/usr/bin`, `/bin`, `/usr/sbin`, `/sbin`, `/opt/homebrew/bin`, `/usr/local/bin`) or an organism `/shims` / `/node_modules/.bin` entry (with `/usr/bin` and `/bin` present) | Returns `process.env.PATH` unchanged when any segment is outside the `launchd` / organism set or when standard directories are already present | JSDoc on `normalizeAmbientPath` and `snapshotAmbientProcessEnvironment` | `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` |
| `generateAntigravityMcpFiles` | `ai/services/fleet/generateAntigravityMcpFiles.mjs:34` | Emits `.neo-fleet-mcp/<key>.sh` with `PATH` carrying the normalized `processEnvironment.PATH` under `exec /usr/bin/env -i` | Refuses non-allowlisted env slots | JSDoc on `generateAntigravityMcpFiles` | `test/playwright/unit/ai/services/fleet/generateAntigravityMcpFiles.spec.mjs` |

Decision Record impact: none

## Acceptance Criteria

- [ ] AC-1: When `process.env.PATH` is `/usr/bin:/bin:/usr/sbin:/sbin` (macOS GUI launch), `FleetLifecycleService.snapshotAmbientProcessEnvironment()` includes `/opt/homebrew/bin` and `/usr/local/bin` after the existing `PATH` segments without duplicating already-present segments.
- [ ] AC-2: `.neo-fleet-mcp/github-workflow.sh` generated from a GUI-launched Fleet start context preserves `/opt/homebrew/bin` in its `env -i` `PATH` so `github-workflow/mcp-server.mjs` resolves `gh`.
- [ ] AC-3: Unit tests in `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` cover minimal Finder `PATH` normalization and deduplication.

## Out of Scope

- Removing `exec /usr/bin/env -i` from `generateAntigravityMcpFiles.mjs` (environment isolation remains required so ambient shell credentials cannot compete with `McpLaunchAdmissionService`).
- Changing `McpLaunchAdmissionService` grant redemption.

## Avoided Traps

- Dropping `-i` from `/usr/bin/env -i` in `.neo-fleet-mcp/<key>.sh`: that would leak arbitrary ambient shell variables and credentials across the profile invoker boundary.
- Prepending `/opt/homebrew/bin` ahead of `shims/`: packaged `Neo Harness.app` relies on `shims/node` coming first in `PATH`; standard system/Homebrew directories must be appended after existing segments.

## Related

- `neomjs/neo-agent-brain#909`
- `neomjs/neo-agent-brain#1035`

unowned-rationale: Filed as a concrete friction capture from the `@neo-gemini-pro` seat restart while completing `#1056` / `neomjs/neo-agent-institution#683`; ready for immediate pickup.

Sweeps (`2026-10-15T23:18Z`):
- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` and `neomjs/neo-agent-institution` at `2026-10-15T23:18Z`; no equivalent found.
- A2A in-flight claim sweep: checked `list_messages({status: 'all', limit: 15})`; no competing claim on Antigravity MCP `PATH`.
- Memory Core rationale sweep: `query_raw_memories` (`"stdioToStreamableHttp sse-disconnect exit status 1 generateAntigravityMcpFiles PATH gh"`); no prior ticket or conflicting decision.
- Own-assignment sweep: only `#1056` assigned to `@neo-gemini-pro`.
- Structure map: `ai/services/fleet/` owns `FleetLifecycleService.mjs` and `generateAntigravityMcpFiles.mjs`.

Origin Session ID: 14f10506-4e8f-40bb-96be-69dac6effa98
Retrieval Hint: "generateAntigravityMcpFiles snapshotAmbientProcessEnvironment /opt/homebrew/bin/gh env -i PATH"



## Timeline

- 2026-10-15T23:24:20Z @neo-gemini-julia added the `bug` label
- 2026-10-15T23:24:20Z @neo-gemini-julia added the `ai` label
- 2026-10-15T23:24:20Z @neo-gemini-julia added the `model-experience` label
- 2026-10-15T23:24:20Z @neo-gemini-julia added the `agent-os` label
- 2026-10-15T23:40:11Z @neo-gpt-sophie cross-referenced by #12
- 2026-10-16T14:12:29Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-16T14:22:49Z @neo-gemini-julia cross-referenced by PR #1065
- 2026-10-16T14:48:00Z @neo-gemini-julia referenced in commit `7dd9621` - "fix(fleet): pin negative-arm PATH normalization in FleetLifecycleService spec (#1061)"
- 2026-10-16T15:31:21Z @tobiu closed this issue
- 2026-10-16T15:31:21Z @tobiu referenced in commit `924b8bd` - "fix(fleet): preserve Homebrew and local bin PATH in Antigravity MCP invokers (#1061) (#1065)

* fix(fleet): preserve Homebrew and local bin PATH in Antigravity MCP invokers (#1061)

* fix(fleet): pin negative-arm PATH normalization in FleetLifecycleService spec (#1061)"

