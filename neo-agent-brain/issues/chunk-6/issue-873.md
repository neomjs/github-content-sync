---
id: 873
title: GitHub Workflow MCP client launches in the caller's checkout
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-10-05T10:16:28Z'
updatedAt: '2026-10-05T13:24:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/873'
author: neo-gpt
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
closedAt: '2026-10-05T13:24:52Z'
---
# GitHub Workflow MCP client launches in the caller's checkout

## Context

While re-reviewing Institution #560, the built-in MCP CLI invoked `npm run ai:mcp-server-github-workflow` from the Engine checkout and failed with `Missing script: "ai:mcp-server-github-workflow"`. The script belongs to the Brain package. Calling the absolute Brain server entrypoint with the same caller environment allowed the managed review to complete; this is a measured launcher failure, separate from a later filesystem-permission refusal.

## The Problem

`ai/mcp/client/config.mjs` leaves the built-in GitHub Workflow entry's `cwd` unset. `Client.loadServerConfig` retains that null, and `createTransport` gives the SDK no child directory. An absolute CLI/config import therefore still makes npm search the caller's package.

The existing `McpClientTransportConfig.spec.mjs` explicitly expects null/undefined for this entry. It protects the failure instead of testing a foreign caller.

## The Architectural Reality

The same canonical config already gives Neural Link a module-derived package root, validated against its owning package manifest. The transport supports `cwd` already; this needs no new service or resolver.

Design authority: the config's module-root JSDoc says the module location is stable while GUI/absolute-config caller directories are not. The observed npm failure independently falsifies caller-owned spawning.

Brain #16 owns native Codex-template materialization, which does not read this client config. Its explicit client/harness boundary makes this a sibling, not a second copy of its work. Own #84's MCP-default ledger covers HTTP admission and transport disposition, not this spawn-directory omission.

## The Fix

Give the built-in GitHub Workflow stdio entry a validated, module-owned spawn directory using the existing package-root pattern. Update the existing transport-config spec with the foreign-caller control. Keep the selected environment, GH_TOKEN requirement, explicit per-request repository and external connection-config overrides intact.

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| built-in `github-workflow.cwd` | client config's module-root ownership | npm resolves the Brain-owned server script from its package | foreign caller never becomes the default spawn root; missing owner/script fails loudly | config JSDoc | foreign-cwd subprocess + transport params |
| `Client#createTransport` stdio parameters | existing SDK handoff | carries the resolved cwd and existing env/argv | explicit external server configuration retains its chosen cwd | existing JSDoc | existing override controls |

Decision Record impact: aligned-with ADR 0019; no new config leaf or competing env resolver.

## Acceptance Criteria

- [ ] The built-in entry resolves an absolute package-owned cwd, independent of an Engine or temporary-directory caller; that package declares the requested script.
- [ ] The old null-cwd expectation becomes a red-before/green-after foreign-caller control, and the actual stdio transport receives the owner cwd.
- [ ] Existing external-config cwd, argv, credential/environment and explicit repository behavior remain covered; no live plane or forge mutation is needed for the regression test.
- [ ] A readonly CLI call from a foreign checkout reaches GitHub Workflow instead of failing npm script resolution, on a profile with the existing required credentials and runtime placement.

## Out of Scope

Codex/Claude/other native harness templates; Neural Link's already-pinned entry; other server entries; server authorization; plane/data-root placement; sandbox permissions; deployment changes.

## Avoided Traps

Hardcoded machine paths or changing the caller's global cwd; adding the removed Brain script to Engine; silently rewriting external configurations; broadening a measured launcher defect into a new runtime topology.

## Related

#16 (harness-owned sibling) · #84 (MCP transport/default ledger) · neomjs/neo#17607 / PR neomjs/neo#17610 (module-root precedent).

## Creation Evidence

Structure map: `npm run ai:structure-map -- --files --loc`, exit 0; existing owner `ai/mcp/client`, four files. No new file is prescribed. ADR 0019 read in full. Live canonical config confirms the omission; the existing spec and package manifest were read. KB surfaced #16 but conflated its scope; the primary body preserves the distinction. MC symptom query returned four low-relevance historical rows. Own-assignment bodies #84 and #90 inspected, no equivalent close target.
Live latest-open sweep: latest 20 open Brain issues (number, author, labels, URL) checked immediately before creation at 2026-10-05 10:16:17 UTC; no equivalent. All-status A2A: newest 30 messages checked in the same window, no overlapping claim. A fresh MC symptom query again returned historical tool-attachment noise, not an equivalent decision.

Origin Session ID: 01a10b58-5990-75e1-ac98-d35d37bb67f5
Retrieval Hint: MCP CLI GitHub Workflow missing npm script from Engine caller cwd; built-in config versus native harness template.


## Timeline

- 2026-10-05T10:16:28Z @neo-gpt assigned to @neo-gpt
- 2026-10-05T10:16:30Z @neo-gpt added the `bug` label
- 2026-10-05T10:16:30Z @neo-gpt added the `ai` label
- 2026-10-05T10:16:31Z @neo-gpt added the `model-experience` label
- 2026-10-05T10:16:31Z @neo-gpt added the `agent-os` label
- 2026-10-05T13:08:04Z @neo-gpt cross-referenced by PR #887
- 2026-10-05T13:24:52Z @tobiu referenced in commit `8b0e239` - "fix(mcp): resolve GitHub Workflow from its owning package (#873) (#887)"
- 2026-10-05T13:24:52Z @tobiu closed this issue

