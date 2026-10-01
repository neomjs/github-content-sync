---
id: 670
title: MCP declarations fail only at Start; a tokenless seat acts as the gh keyring
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T12:14:43Z'
updatedAt: '2026-10-01T12:28:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/670'
author: neo-opus-grace
commentsCount: 0
parentIssue: 659
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T12:28:32Z'
---
# MCP declarations fail only at Start; a tokenless seat acts as the gh keyring

## Context

Split from #659, the operator's MCP-toggle request. #659 carries five fixes, and the Claude half of them waits on #669 (the Code-tab carrier). Fix 3 and Fix 5 need nothing from #669, so this ticket carries them and one PR resolves it. #659 keeps Fix 2, Fix 4, AC-2 to AC-5 and AC-8.

## The Problem

1. **The registry stores what Start refuses.** `FleetRegistryService.defineAgent` and `configureAgent` normalize MCP overrides and write the registry without asking the plan. `createManagedAgentWorkspacePlan` then refuses at the next Start. Turning `github-workflow` on for a `claude-desktop` seat is accepted by the cockpit card, and the next Start fails with "Claude Desktop cannot represent startup-required Fleet secret env…"; the card shows no reason. The registry also repeats one of the plan's checks (tenant grammar) in its own words.
2. **A seat can act as the host's account.** `GraphqlService.#getAuthToken` falls back to `gh auth token` when no token env is set. On a Fleet host the keyring is the operator's, so a seat whose `GH_TOKEN` never arrived (Claude Desktop strips its MCP children's environment, #669) reads GitHub as the operator. Writes stay behind the identity assertion; reads do not.

## The Architectural Reality

- `ai/services/fleet/managedAgentWorkspacePlan.mjs` `assertLogicalHarnessSupported`: two harness-level refusals (no launch adapter, Antigravity) and three declaration refusals (tenant grammar, catalog-unsupported server, Claude Desktop secret env).
- `ai/services/fleet/FleetRegistryService.mjs` `defineAgent` / `configureAgent`. `FleetControlBridge` turns a `FleetRegistryService.<method>:` error into `{status: 'rejected', reason}`.
- `ai/services/github-workflow/GraphqlService.mjs` `#getAuthToken`. `HealthService.checkAgentIdentity` and the tool layer's identity assertion read `NEO_AGENT_IDENTITY` from the environment.

## The Fix

1. The plan exports its declaration refusals as `mcpDeclarationRefusal({harnessType, mcpMatrix, tenant})` and throws with it; `defineAgent` and `configureAgent` reject with it, replacing the registry's own tenant check.
2. `#getAuthToken` refuses the CLI fallback when `NEO_AGENT_IDENTITY` is set and neither `GH_TOKEN` nor `GITHUB_TOKEN` is, and names the seat.

## Acceptance Criteria

- [ ] AC-1 (#659's AC-6): `defineAgent` and `configureAgent` reject a declaration the plan would refuse, with the plan's reason after the method prefix, and write nothing; one arm per declaration refusal class.
- [ ] AC-2 (#659's AC-7): with `NEO_AGENT_IDENTITY` set and no token env, `GraphqlService` throws a named error and never runs `gh auth token`.

## Out of Scope

- The plan's harness-level refusals in the registry: an external seat of an unlaunchable harness is a legal row, and Start keeps refusing it.
- The other `gh` CLI calls in the github-workflow server (health, PR diff): they honour the same `GH_TOKEN`, and writes stay behind the identity assertion.
- Everything #659 keeps.

## Decision Record impact

`aligned-with ADR 0019`: no leaf is added; `NEO_AGENT_IDENTITY` is read the way the server's other identity readers read it.

## Related

Parent #659. #669 deletes the Claude Desktop branch from `mcpDeclarationRefusal` once those rows become representable. #571.

## Sweeps

Live latest-open sweep: latest 20 open Brain issues at 2026-10-01T12:13:27Z, no equivalent (#659 is the parent). A2A in-flight sweep (30 newest, latest 12:09Z): Euclid's #669 boundary message leaves this slice with #659's owner; no competing claim. MC sweep: "the toggle persists but the Start refuses it; … acts as the host keyring account", 6 results, no prior decision beyond #659's filing. Own-assignment sweep: #659, the parent, only. Structure map: exit 0; owning folders `ai/services/fleet` and `ai/services/github-workflow`; no new file.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "registry refuses an MCP declaration with the plan's reason mcpDeclarationRefusal; GraphqlService refuses the gh keyring for a seat"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-10-01T12:14:43Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T12:14:44Z @neo-opus-grace added the `bug` label
- 2026-10-01T12:14:45Z @neo-opus-grace added the `ai` label
- 2026-10-01T12:14:45Z @neo-opus-grace added the `security` label
- 2026-10-01T12:14:45Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T12:14:49Z @neo-opus-grace added parent issue #659
- 2026-10-01T12:14:54Z @neo-opus-grace cross-referenced by #659
- 2026-10-01T12:15:04Z @neo-opus-grace referenced in commit `704d677` - "fix(fleet): the registry refuses an MCP declaration its harness cannot carry, and a seat never borrows the gh keyring (#670)

defineAgent and configureAgent ask the plan's own MCP refusals (mcpDeclarationRefusal) before
they store a declaration, so Start no longer refuses what the cockpit just accepted. The two
registry-local tenant checks are gone; the plan's check covers them.

GraphqlService refuses the `gh auth token` fallback when NEO_AGENT_IDENTITY names a seat and no
token env is set: the host keyring's account would otherwise act for the seat.

The isolated-child cache arm imported `<brain>/src/Neo.mjs`, which the Brain does not have; it
now imports the engine package and runs again."
- 2026-10-01T12:15:25Z @neo-opus-grace cross-referenced by PR #671
- 2026-10-01T12:19:01Z @neo-opus-grace cross-referenced by #672
- 2026-10-01T12:28:32Z @tobiu referenced in commit `d68da3b` - "fix(fleet): the registry refuses an MCP declaration its harness cannot carry, and a seat never borrows the gh keyring (#670) (#671)

defineAgent and configureAgent ask the plan's own MCP refusals (mcpDeclarationRefusal) before
they store a declaration, so Start no longer refuses what the cockpit just accepted. The two
registry-local tenant checks are gone; the plan's check covers them.

GraphqlService refuses the `gh auth token` fallback when NEO_AGENT_IDENTITY names a seat and no
token env is set: the host keyring's account would otherwise act for the seat.

The isolated-child cache arm imported `<brain>/src/Neo.mjs`, which the Brain does not have; it
now imports the engine package and runs again."
- 2026-10-01T12:28:32Z @tobiu closed this issue

