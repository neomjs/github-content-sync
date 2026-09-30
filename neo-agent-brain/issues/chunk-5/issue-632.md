---
id: 632
title: Fleet's Codex CLI default points at a removed bundle entry
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T12:47:04Z'
updatedAt: '2026-09-30T14:43:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/632'
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
closedAt: '2026-09-30T14:29:05Z'
---
# Fleet's Codex CLI default points at a removed bundle entry

## Context

The installed FM first-start witness for the new Codex Desktop seat failed on 2026-09-30 at `FleetLifecycleService.assertRemoteMcpCapability`: the configured `ChatGPT.app/Contents/Resources/codex` does not exist. The current app ships `Resources/codex-cli/codex-package.json`, whose `entrypoint` is `bin/codex`; that wrapper exists and reports `codex-cli 0.159.2`.

## The Problem

The bundled CLI moved, but the Fleet binary default did not. Codex Desktop uses this CLI to validate its remote MCP grammar before launch, so a working desktop executable is refused before any new seat starts.

## The Architectural Reality

`ai/configBase.mjs` owns `fleet.harnessBinaries.codex` and its `NEO_FLEET_CODEX_BIN` binding. `FleetLifecycleService.getHarnessBinaryPath()` consumes that resolved leaf for CLI launch and Codex Desktop validation. An isolated process supplied the manifest-declared path through the existing environment binding; the real `assertRemoteMcpCapability()` passed and returned the installed desktop executable too. No adapter or permission bypass is required.

## The Fix

Update the existing default to `/Applications/ChatGPT.app/Contents/Resources/codex-cli/bin/codex`, the package's wrapper entrypoint. Refresh the adjacent path/channel prose. Preserve the environment override and the capability admission gate; older or custom installations continue to declare their own path.

## Contract Ledger

| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Codex CLI location | `fleet.harnessBinaries.codex` | Default names the installed package entrypoint | Existing explicit override and unavailable-binary refusal remain | Leaf JSDoc and launch contract prose | Installed manifest, version and real Fleet capability probe |

## Decision Record impact

Aligned with ADR-0019: change the leaf default, retain its environment binding, and keep consumers unchanged. The ADR was read before this config touch.

## Acceptance Criteria

- [ ] AC-1: The default Codex CLI location matches the installed package's declared wrapper entrypoint and serves both CLI and Desktop consumers.
- [ ] AC-2: Explicit `NEO_FLEET_CODEX_BIN` overrides and existing remote-MCP capability refusal remain intact.
- [ ] AC-3: Config validation and relevant existing lifecycle/launch tests pass; an installed capability probe accepts the new default.

## Out of Scope

OpenAI app modification, filesystem symlinks, fallback binary searches, provider login, native plane-admission UI and complete first-session acceptance.

## Related

#16 · #571 · neomjs/neo-agent-institution#12

Sweeps: latest 20 open Brain issues plus recent all-state A2A, exact binary search, and own assignments (three unrelated open tickets) show no overlapping leaf. Three raw-memory framings for the unavailable bundled CLI returned no relevant prior mapping. Existing `ai/services/fleet` consumer and `ai/configBase.mjs` declaration are the owners; the structure map was already run during this onboarding lane. Installed first-turn acceptance stays under Institution #12.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e.
Retrieval Hint: Codex Desktop Resources/codex unavailable codex-cli/bin/codex.

## Timeline

- 2026-09-30T12:47:04Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T12:47:05Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T12:47:05Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T12:47:06Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T12:54:00Z @neo-gpt-emmy cross-referenced by PR #633
- 2026-09-30T13:08:45Z @tobiu referenced in commit `5153a4b` - "Merge pull request #633 from neomjs/codex/632-codex-cli-path

fix(fleet): use the packaged Codex CLI entrypoint (#632)"
- 2026-09-30T13:25:20Z @neo-gpt-emmy cross-referenced by #635
- 2026-09-30T14:29:05Z @tobiu closed this issue
- 2026-09-30T14:35:13Z @neo-gpt-emmy cross-referenced by #639
### @neo-gpt-emmy - 2026-09-30T14:43:24Z

**Installed verification, 2026-09-30.** The full FM package at Brain `6a714ae` passes the real remote-MCP capability gate with the corrected bundled CLI `codex-cli/bin/codex`. The operator then started the new Codex Desktop seat through FM, logged in, and supplied a completed first-turn screenshot. This discharges the CLI-location leaf. The distinct local MCP execution-mode issue is #639; full onboarding acceptance remains neomjs/neo-agent-institution#12.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e

- 2026-09-30T15:08:28Z @neo-fable-clio cross-referenced by PR #363
- 2026-09-30T15:40:17Z @neo-opus-grace cross-referenced by PR #640

