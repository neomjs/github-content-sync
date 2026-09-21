---
id: 330
title: Expose tenant-target support in the public harness catalog
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-05T13:05:18Z'
updatedAt: '2026-09-05T14:20:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/330'
author: neo-gpt-emmy
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
blocking:
  - '[x] 43 Consume Brain Fleet contract and delete vocabulary twins'
closedAt: '2026-09-05T14:20:49Z'
---
# Expose tenant-target support in the public harness catalog

## Context

Institution #43 is adopting the public Fleet contract delivered by merged #326. A current-source sweep found that `AgentConfigComponent.mjs:198,326` still consumes `supportsTenantMcpTarget` to offer and submit remote tenant choices. The producer kept that helper private, so deleting the local twin either breaks the card or imports a private module into its worker.

This is a new, linked successor to completed #217; that ticket is not reopened.

## The Problem

The original boundary grouped two different things together: **whether a harness configuration grammar can represent remote HTTP MCP targets** is catalog capability vocabulary; **normalizing a target, resolving credential slots and authorizing access** are private trust decisions.

At the merged producer, `ai/services/fleet/mcpServers.mjs:15–22,79–81` answers the capability question only by membership in a frozen six-harness list. Institution consumes no target-normalization or credential-environment helper in its views. The public harness catalog currently carries type/label only.

## The Architectural Reality

- `src/fleet/contract/harnessTypes.mjs` is the canonical public harness registration and caller-owned list/lookup implementation.
- `src/fleet/contract/index.mjs` exports that module through `neo-agent-brain/fleet-contract`; its dependency-free graph is tested in `test/playwright/unit/src/fleet/contract/fleetContract.spec.mjs`.
- `FleetRegistryService.mjs` and `managedAgentWorkspacePlan.mjs` consume the existing support predicate for server-side validation.
- The private `TENANT_MCP_HARNESS_TYPES` list is currently used only by that predicate. Preserve one registration source, not an unused compatibility list.

## The Fix

Add a Boolean `tenantMcpTarget` capability to each canonical harness record and expose `supportsTenantMcpTarget(type)` from that same module. Preserve the current supported set and unknown-type refusal. Re-point Brain consumers to the public capability helper, and remove the displaced private helper/list when no consumer remains.

Keep `normalizeMcpTarget`, credential environment names and all bearer/authorization/ownership behavior private. No new package or runtime module is needed. Institution #43 then consumes this field/helper through its pinned public entry and deletes the last local twin.

## Contract Ledger

| Surface | Authority | Behavior | Fallback / boundary | Docs | Evidence |
|---|---|---|---|---|---|
| Harness records | Existing `HARNESS_TYPES` registration | `tenantMcpTarget: Boolean` per entry; list/lookup copies preserve it | Existing type/label/order unchanged | Catalog JSDoc | Exact capability matrix and caller-owned-copy tests |
| Public support query | Catalog flag | `supportsTenantMcpTarget(type)` reads the canonical record | Unknown/malformed type returns false; no environment or service reads | Helper JSDoc/README | Public-entry calls and browser graph/bundle |
| Brain validation | Existing registry/workspace-plan consumers | Read the same public capability fact | Target normalization and authorization stay private | Import/ownership documentation | Existing target-validation tests |
| Private target module | Current `mcpServers.mjs` | Retain normalization and credential policy only | No public graph edge to private module; no duplicate capability registry | Module JSDoc | Private-export negatives and import census |

## Decision Record impact

Aligned-with ADR 0038 §2.8: capability vocabulary crosses; trust decisions do not. This corrects the overdrawn capability classification in #217, without changing grant or credential authority.

## Acceptance Criteria

- [ ] Every public harness record declares the Boolean capability; the currently supported six types remain supported and the other registered/unknown types refuse.
- [ ] The public entry exports the support query over that catalog; returned catalog records remain caller-owned and the source remains frozen.
- [ ] Both Brain consumers use the canonical query; the former private helper/list has no duplicate or compatibility copy.
- [ ] Target normalization, credential slots and authorization behavior remain private and existing validation tests pass.
- [ ] Public import/graph/browser-bundle checks still reach only the dependency-free contract; private policy exports remain absent.
- [ ] README/JSDoc state capability versus policy, and Institution #43 receives the immutable producer handoff.

## Out of Scope

Institution implementation, package restructuring, new Fleet verbs, credentials/grants, target-normalization changes, or changing which harnesses support tenant targets.

## Avoided Traps

Exporting the entire private MCP module leaks a runtime boundary. Keeping a client list preserves the drift the canonical contract removes. A second exported support list also duplicates registration; use a flag on the existing catalog and derive any genuinely retained consumer view from it.

## Related

#217 · #326 · neomjs/neo-agent-institution#43

Live latest-open sweep: latest 20 Brain issues and open PRs checked immediately before creation; no equivalent. All-status A2A claim sweep found the peer boundary discussion but no competing implementation claim. Own-assignment sweep: one unrelated open issue (#48). MC problem-noun searches returned unrelated rows; KB synthesis is unavailable, so current producer/consumer source and the reviewed boundary supply the evidence. Structure-map ran; the existing `src/fleet/contract` catalog is the owner, with private consumers in `ai/services/fleet`.

Origin Session ID: 0e9cccb4-3c5c-4f04-be91-badb29c50238

Retrieval Hint: `AgentConfigComponent supportsTenantMcpTarget public harness capability private target normalization`.


## Timeline

- 2026-09-05T13:05:18Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-05T13:05:19Z @neo-gpt-emmy added the `bug` label
- 2026-09-05T13:05:19Z @neo-gpt-emmy added the `ai` label
- 2026-09-05T13:05:19Z @neo-gpt-emmy added the `refactoring` label
- 2026-09-05T13:05:19Z @neo-gpt-emmy added the `testing` label
- 2026-09-05T13:05:20Z @neo-gpt-emmy added the `architecture` label
- 2026-09-05T13:05:20Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-05T13:19:15Z @neo-gpt-emmy cross-referenced by PR #331
- 2026-09-05T14:03:22Z @neo-gpt-emmy cross-referenced by #43
- 2026-09-05T14:06:28Z @neo-opus-ada cross-referenced by PR #118
- 2026-09-05T14:20:49Z @tobiu referenced in commit `bd54171` - "Merge pull request #331 from neomjs/codex/330-harness-tenant-capability

feat(fleet): expose harness tenant-target capability (#330)"
- 2026-09-05T14:20:49Z @tobiu closed this issue

