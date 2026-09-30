---
id: 630
title: Codex provisioning reads a template the Brain does not ship
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T11:02:29Z'
updatedAt: '2026-09-30T11:45:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/630'
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
closedAt: '2026-09-30T11:45:51Z'
---
# Codex provisioning reads a template the Brain does not ship

## Context

The operator-directed first onboarding through installed Fleet Manager on 2026-09-30 reached canonical Add Agent readback for a Codex Desktop seat, then Start failed before spawning the harness. The shell log at 10:58:04Z reports `FLEET_WORKSPACE_PREPARATION_FAILED`, caused by ENOENT reading `<agentosRuntimeRoot>/.codex/config.template.toml` in `prepareCodexArtifacts`.

Installed Brain revision: `9f42809`. The same read remains on current Brain source, whose tracked tree also contains no `.codex` template.

## The Problem

The workspace preparer still depends on a manual Codex setup template outside the Brain's shipped runtime. The Engine retains that template for its own manual setup. Copying it into an installed app by hand would hide the product failure and duplicate unrelated project/model policy.

The existing workspace spec manufactures the missing runtime template in every fixture, so its Codex coverage cannot expose this failure.

## The Architectural Reality

- `ai/services/fleet/prepareManagedAgentWorkspace.mjs:703-709` reads the runtime template before emitting project MCP tables.
- `renderCodexProjectConfig` replaces the template's Neo MCP sections with the already-resolved managed server plan.
- `projectCodexOwnedProjection` and `convergeTransportArtifact` already limit Fleet ownership to its `neo-mjs-*` tables and preserve unrelated existing policy.
- Claude workspace generation already emits its managed MCP configuration from the plan without a repo-root template.
- Structure-map command passed; the existing `ai/services/fleet` artifact composer owns this repair. No new module, config leaf or service is needed.

## The Fix

Generate the Codex project's Fleet-owned MCP tables directly from the resolved plan. Remove the runtime-template read and obsolete template-splicing logic from initial generation. Retain the existing convergence/transport migration rules for resident-owned settings and edited managed tables. Stop manufacturing a template in the common test fixture; prove both Codex adapters work with a template-free runtime.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Failure/fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Codex project config | `prepareCodexArtifacts`, resolved MCP plan | Emit the managed tables without a runtime-root template | Existing path and divergence failures remain | Renderer JSDoc | Fresh Codex and Codex Desktop preparation without a template |
| Existing resident config | `projectCodexOwnedProjection`, `convergeTransportArtifact` | Preserve unrelated policy and current transport migration | Edited owned tables still refuse | Existing convergence contract | Re-entry, policy preservation and divergence tests |
| Isolated Codex home | `renderCodexHomeConfig` | Existing auth-store/memory policy unchanged | Existing file guards remain | Existing JSDoc | Current home assertions |

## Decision Record impact

None: restores the existing Brain-owned artifact contract; changes no AiConfig resolution or template ownership. The Engine's manual setup template remains its own surface.

## Acceptance Criteria

- [ ] AC-1: Both `codex` and `codex-desktop` prepare successfully with no runtime-root `.codex/config.template.toml`, emitting their complete resolved MCP plans.
- [ ] AC-2: Existing resident policy and unrelated MCP tables survive re-entry and supported transport migration; edited Fleet-owned entries retain refusal.
- [ ] AC-3: The Codex home remains isolated with its existing auth-store and memory settings; generated artifacts remain secret-free.
- [ ] AC-4: CI-covered workspace fixtures no longer synthesize the missing runtime dependency and include the failing-before/fixed-after first-preparation case.

## Out of Scope

Seat-directory migration, provider login, roster seeding, wake arming, Neural Link permissions, and general UI error-copy work. Installed first-turn acceptance remains under neomjs/neo-agent-institution#12; this leaf repairs the prerequisite, not the whole journey.

## Avoided Traps

No manual template plant in the app bundle, Engine template copy, model-specific default injection, or fallback search through arbitrary checkouts. Preserve resident files through the existing narrow ownership primitive.

## Related

#571 · #16 · #628 · neomjs/neo-agent-institution#12

Latest-open sweep: latest 20 Brain issues and latest 30 all-state A2A rows checked immediately before filing on 2026-09-30; no equivalent claim. `#16` is the separate manual-template cwd defect. Own assignments: four open; `#628` body read, owns NL projection only. MC searches for first Start/missing template and split ownership found no prior mapping of this defect; earlier session `f50c5803-05f6-47d5-a759-b6d123405758` supplies the operator's real FM-first-boot sequence. This installed blocker promotes defect observation `c3f0c1254e87dd7b`.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e.
Retrieval Hint: Codex first Start missing config.template.toml; prepareCodexArtifacts.

## Timeline

- 2026-09-30T11:02:29Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T11:02:30Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T11:02:30Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T11:02:30Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T11:08:59Z @neo-gpt-emmy cross-referenced by PR #631
- 2026-09-30T11:29:23Z @tobiu referenced in commit `8a42800` - "Merge pull request #631 from neomjs/codex/630-codex-template

fix(fleet): prepare Codex without an unshipped template (#630)"
### @neo-gpt-emmy - 2026-09-30T11:45:50Z

Completed by merged PR #631 (8a42800). Installed first-boot acceptance remains under neomjs/neo-agent-institution#12.

- 2026-09-30T11:45:52Z @neo-gpt-emmy closed this issue

