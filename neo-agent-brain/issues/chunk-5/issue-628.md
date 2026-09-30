---
id: 628
title: FM-launched agents are forced into read-only Neural Link
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T10:00:41Z'
updatedAt: '2026-09-30T11:45:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/628'
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
closedAt: '2026-09-30T11:45:49Z'
---
# FM-launched agents are forced into read-only Neural Link

## Context

The operator directed on 2026-09-30 that Fleet Manager must launch agents with writable Neural Link access. A new maintainer started through FM should retain the live application tools available to the same harness started directly.

The historical implementation deliberately imposed the opposite policy: [neomjs/neo#13121](https://github.com/neomjs/neo/issues/13121) equated every FM-spawned agent with a restricted embedded agent. This ticket corrects that launch-time classification under the current operator direction.

## The Problem

FM removes the agent's ability to operate the live application merely because FM started its harness. This is enforced in two independent paths, so removing only the process environment assignment does not repair generated Claude Desktop MCP configuration.

This is a policy correction, not a claim that the original implementation accidentally violated its ticket.

## The Architectural Reality

- `ai/services/fleet/FleetLifecycleService.mjs:255,508-513` defaults `toolProjectionMode` to `harness-embedded` and injects `NEO_NL_TOOL_PROJECTION_MODE` into every child.
- `ai/services/fleet/managedAgentWorkspacePlan.mjs:51-60` makes that variable part of the Neural Link runtime/required environment contract.
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs:840-841` writes `harness-embedded` directly into generated Claude Desktop configuration.
- `ai/mcp/server/neural-link/openapi.yaml:5-15` gives that projection only the read tier; `ToolService.isToolAllowedForProjection()` enforces it for tool visibility and calls.
- The ordinary developer path already exists: the Neural Link server resolves an absent forced mode to no projection ceiling. Explicit restricted server projections are a separate supported contract.
- Structure map: `npm run ai:structure-map -- --files --loc` passed. The owning consumer is the existing `ai/services/fleet` launch/workspace composition; no new module or service is needed.

## The Fix

Remove the automatic FM launch-to-embedded classification from the spawner and generated managed-workspace projections. FM-launched agents use the ordinary Neural Link developer surface. Retire the redundant variable requirement and stale prose at those consumers. Existing generated artifacts containing an explicit projection retain the divergence guard; their retired Fleet-owned projection entry must be reconciled during upgrade, not silently overwritten.

Preserve the explicit restricted projection mechanism in the Neural Link server, its list/call enforcement, authenticated identity, Bridge-token handling, and credential/environment isolation. Do not widen the shared `harness-embedded` profile, create a special case for a named maintainer, or add a new launch-permission switch.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / failure | Docs | Evidence |
|---|---|---|---|---|---|
| FM child environment | `FleetLifecycleService.start()`; operator direction above | FM does not force a read-only NL mode on its agent | Existing identity, credential-slot and launch validation remain | Class and spawn JSDoc | Production-bound launch test proves mutation tools remain available through the resulting NL context |
| Generated MCP configuration | `managedAgentWorkspacePlan` NL row and `renderClaudeJsonContent()` | No hardcoded/read-only projection requirement in generated FM seat artifacts | Preparation errors remain explicit; no credential leakage | Producer JSDoc and existing assertions | Generated artifacts for relevant harness families contain no FM-imposed ceiling |
| Explicit restricted NL contexts | `BaseServer.getToolProjectionContext()`, `ToolService.isToolAllowedForProjection()`, OpenAPI policy | Explicit embedded/probe modes keep their restricted lists and call checks | Unknown modes remain refused | Existing server contract | Existing restricted-mode controls stay green; positive developer mutation/list control is added or retained |
| Maintainer onboarding witness | Existing FM add/start path | A managed seat can use a real NL mutation on an isolated test target | A registered row or launched process alone is not proof | PR evidence declaration | State the achieved evidence level; do not claim installed-product acceptance from unit tests |

## Decision Record impact

Aligned with ADR 0020's operate-your-fleet and live-app co-habitation product, and the trusted-development world recorded in #141. This reverses the narrower consumer premise of `neomjs/neo#13121`; it does not redefine the explicit embedded projection, Bridge authentication, or multi-writer coordination policy under #143.

The implementation is a bounded, reversible consumer correction. No new protocol, capability class, or server-wide projection policy is introduced.

## Acceptance Criteria

- [ ] FM spawn and managed-workspace generation both stop imposing `harness-embedded` on launched agents.
- [ ] Tests exercise the actual launch/preparation outputs and show the resulting default NL surface admits a mutation-class operation, not just a changed string.
- [ ] Explicit embedded/probe projections still hide/refuse excluded tools; unknown projection modes still fail closed.
- [ ] Identity injection, separate credential classes, environment validation and secret-free stored/generated surfaces remain covered.
- [ ] Obsolete FM-spawned-equals-read-only assertions and JSDoc are corrected at the changed consumers.
- [ ] The PR distinguishes tested launch/projection behavior from any installed FM acceptance still requiring a real target.

## Out of Scope

New lock/authorization systems; changing deployed-cloud application grants; removing explicit restricted modes; changing account/login or wake adapters; named-agent exceptions; deployment or human merge.

## Avoided Traps

- Changing only the parent env leaves the generated Claude Desktop ceiling in place.
- Widening `harness-embedded` globally changes unrelated callers.
- A per-name exception hides the incorrect FM classification.
- A new permission toggle preserves the obsolete restriction as product complexity without a current requirement.

## Related

#143 · #141 · neomjs/neo#13121 · neomjs/neo#13106 · neomjs/neo#13084

Live latest-open sweep: latest 20 Brain issues read immediately before filing on 2026-09-30; no equivalent leaf. Exact org searches recovered the original projection/injection tickets and the broader NL epics, whose bodies were read. A2A all-status latest 30 showed no competing claim on these consumers. MC rationale queries returned irrelevant/stale rows; the original intent was recovered from live issue bodies rather than inferred from search silence. Own-assignment sweep: three open issues (#426, #306, #48), none on this surface.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e

Retrieval Hint: `FleetLifecycleService FM spawned embedded read-only Neural Link generated Claude Desktop projection`



## Timeline

- 2026-09-30T10:00:41Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T10:00:43Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T10:00:43Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T10:00:43Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T10:32:52Z @neo-gpt-emmy cross-referenced by PR #629
- 2026-09-30T10:50:56Z @neo-gpt-emmy referenced in commit `9028f0b` - "docs(neural-link): clarify explicit server projection (#628)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-09-30T10:51:37Z @neo-gpt-emmy referenced in commit `0424d1c` - "docs(neural-link): align projection CLI help (#628)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-09-30T11:02:30Z @neo-gpt-emmy cross-referenced by #630
- 2026-09-30T11:20:19Z @neo-opus-grace cross-referenced by PR #631
- 2026-09-30T11:28:54Z @tobiu referenced in commit `659f210` - "Merge pull request #629 from neomjs/codex/628-fm-nl-tools

feat(fleet): launch agents with writable Neural Link (#628)"
### @neo-gpt-emmy - 2026-09-30T11:45:48Z

Completed by merged PR #629 (659f210). Installed first-boot acceptance remains under neomjs/neo-agent-institution#12.

- 2026-09-30T11:45:49Z @neo-gpt-emmy closed this issue

