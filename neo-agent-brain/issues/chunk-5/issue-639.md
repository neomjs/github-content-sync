---
id: 639
title: Packaged Fleet emits GUI commands for local MCP servers
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T14:35:12Z'
updatedAt: '2026-09-30T16:12:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/639'
author: neo-gpt-emmy
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
closedAt: '2026-09-30T16:12:50Z'
---
# Packaged Fleet emits GUI commands for local MCP servers

## Context

Installed onboarding on 2026-09-30: Institution `54d8ac2`, Brain `6a714ae`, Electron 43.5.0. FM successfully starts the new Codex Desktop process with an isolated profile and working repository. The operator completed provider login. Local MCP readiness is a separate failure.

## The Problem

The generated project TOML names the **Neo Harness GUI executable** as `command` for Neural Link and GitHub Workflow, with the server `.mjs` paths as arguments and no `ELECTRON_RUN_AS_NODE` mode. Launch-time logs show extra shell boot attempts rejecting the already-owned Fleet port. The configured program is executable, but that does not make it a Node invocation.

Remote MC/KB readiness separately returned HTTP 200 and the expected new-seat identity. The macOS browser-data permission notification is separately captured in neomjs/neo-agent-institution#354; granting that permission does not change this executable-mode contract.

## The Architectural Reality

- `prepareManagedAgentWorkspace.mjs` defaults `nodePath` to `process.execPath`, then `bindManagedAgentWorkspacePlan` projects it as the local MCP command.
- The packaged Fleet process runs on Electron in Node mode. Its `process.execPath` is the app executable, whose mode depends on environment.
- `FleetLifecycleService` deliberately forwards a minimal environment to the resident; putting Node mode on the Desktop harness itself would be incorrect.
- The local Codex tables carry command/args and env-variable names; a bare executable check in `assertExecutablePlan` cannot prove that the server will execute as Node.
- Institution has a packaged Node shim, but its current form requires `NEO_HARNESS_ELECTRON_BIN`; that variable is not in the resident's ambient allowlist. Replacing one string with that shim alone is not a proven repair.

## The Fix

Make the bound local MCP invocation carry the execution context required by the selected Node runtime. Keep host-runtime binding at the workspace effect boundary and express Node mode only on the MCP child, never on the resident Desktop process or by copying the parent environment. Preserve explicit Node overrides and ordinary-Node output.

Native harness renderers and the generated-adapter readback must agree on that invocation. Use an actual packaged Electron execution witness, not only file existence or TOML parsing. Reconcile the previously Fleet-generated projection without overwriting resident-authored settings or login data; retain refusal for genuine manual divergence.

## Contract Ledger

| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Bound local MCP command | `bindManagedAgentWorkspacePlan`; selected host runtime | Execute the server as Node in Node- and Electron-hosted Fleet | Unsupported runtime must be named before reporting readiness | Preparation/runtime JSDoc | Native renderer fixtures plus packaged Node-mode execution |
| Resident launch environment | `FleetLifecycleService` allowlist | Desktop remains in GUI mode; MCP-only runtime values stay scoped | No copied parent secrets or global Node-mode setting | Existing environment contract | Distinct Desktop and MCP child assertions |
| Existing generated config | Workspace convergence owner | Update recognized Fleet output while preserving resident additions and login files | Hand-edited managed content still refuses | Convergence docs | Prior-generation fixture and unchanged private-file bytes |

## Acceptance Criteria

- [ ] AC-1: A packaged Electron-hosted Fleet generates local MCP invocations that execute the intended Node server rather than opening another shell.
- [ ] AC-2: Ordinary Node hosting and explicit Node overrides preserve their supported behavior, with no global `ELECTRON_RUN_AS_NODE` added to a Desktop harness.
- [ ] AC-3: Generated native config/readback agrees with the execution context; credential values remain absent from generated artifacts.
- [ ] AC-4: A previously provisioned seat receives the corrected managed projection without losing login data or resident-owned settings; genuine divergence remains refused.
- [ ] AC-5: Record a real packaged-runtime MCP/Node execution receipt; first full peer turn remains neomjs/neo-agent-institution#12.

## Out of Scope

Browser-data/TCC permissions, multi-repository selection, changing provider accounts, code signing, manual config patches that bypass Fleet, and broadening the parent environment allowlist to make a test pass.

## Related

#635 · #632 · #571 · neomjs/neo-agent-institution#12

Sweeps: latest 20 open Brain issues, exact Electron/MCP search, recent all-state A2A, and own-assignment bodies (#635 plus prior CLI scope) show no equivalent repair. #571 owns seat layout, not executable mode. Three memory framings found prior authority that the harness owns stdio children, but no packaged-mode repair. Structure map ran this turn; existing Fleet preparation/lifecycle files own the boundary. Decision Record impact: none anticipated; no AiConfig change is prescribed.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: packaged Fleet process.execPath Electron GUI local stdio MCP Node execution mode.

## Timeline

- 2026-09-30T14:35:12Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T14:35:14Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T14:35:14Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T14:35:14Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T14:43:25Z @neo-gpt-emmy cross-referenced by #632
- 2026-09-30T14:43:27Z @neo-gpt-emmy cross-referenced by #635
- 2026-09-30T15:18:19Z @neo-gpt-emmy cross-referenced by PR #640
- 2026-09-30T15:55:45Z @neo-gpt-emmy referenced in commit `0b8a493` - "fix(fleet): centralize child runtime contract and guard Desktop mode (#639)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-09-30T16:04:35Z @neo-gpt-emmy cross-referenced by #642
- 2026-09-30T16:11:19Z @tobiu referenced in commit `7d5af71` - "Merge pull request #640 from neomjs/codex/639-packaged-mcp-node-mode

fix(fleet): bind MCP children to the host Node runtime (#639)"
### @neo-gpt-emmy - 2026-09-30T16:12:49Z

Delivered by merged PR #640 (`7d5af713d60e6727dca029f618b80b122e746cf4`). Grace's Round-2 approval at `0b8a493` discharges all three actions; the packaged candidate at that head passes source equality, native Codex config readback, MCP initialization and the runtime probe. The Desktop env exclusion also has a failing allowlist-mutation control. The active seat's coordinated update and full attachment checks remain on neomjs/neo-agent-institution#12, as scoped by this issue's AC-5.

- 2026-09-30T16:12:51Z @neo-gpt-emmy closed this issue
- 2026-09-30T16:31:02Z @neo-opus-grace cross-referenced by PR #643

