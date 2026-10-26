---
id: 1265
title: Add MCP Registry server.json manifests for the six MCP servers
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-25T02:12:34Z'
updatedAt: '2026-10-25T23:58:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1265'
author: neo-gemini-julia
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
closedAt: '2026-10-25T03:09:15Z'
---
# Add MCP Registry server.json manifests for the six MCP servers

## Context

Coding agents and MCP clients query the official MCP Registry (`https://registry.modelcontextprotocol.io/v0/servers`) to discover and configure MCP servers. Queries for `neomjs`, `neo-mjs`, `neo.mjs`, and `io.github.neomjs` currently return 0 servers because none of our six MCP servers (`file-system`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `memory-core`, `neural-link`) carry an official MCP Registry `server.json` manifest in `ai/mcp/server/<server>/`.

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-25T02:11:33Z` (`#1258` down to `#957`); no equivalent found.
- A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 10 })` at `2026-10-25T02:11:36Z`; no competing claim on MCP Registry manifests.
- MC sweep: `query_raw_memories({ query: 'registry.modelcontextprotocol.io server.json MCP Registry neomjs/neo-agent-brain' })`, 5 results, no prior decision found.
- Own-assignment sweep: `2` open issues assigned (`#1257`, `#1248`), neither overlapping.
- Structure-map check (`npm run ai:structure-map -- --files --loc`): `ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}` are the six canonical MCP server roots alongside their `openapi.yaml` contracts.

## The Problem

Publishing to the official MCP Registry (`mcp-publisher publish` under `io.github.neomjs/*` via GitHub org namespace authentication) requires a valid `server.json` manifest for each MCP server conforming to `https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`:
- `name` matching `^io\.github\.neomjs/[a-zA-Z0-9._-]+$` (`3..200` chars, reverse-DNS GitHub org namespace)
- `title` (`1..100` chars)
- `description` (`1..100` chars — note the strict `maxLength: 100` constraint in `ServerDetail`)
- `version` matching `openapi.yaml` (`"1.0.0"`, non-range semantic version)
- `websiteUrl`: `"https://neomjs.com"`
- `repository`: `{ "id": "1344062228", "source": "github", "subfolder": "ai/mcp/server/<name>", "url": "https://github.com/neomjs/neo-agent-brain" }`

## The Architectural Reality

Each MCP server lives under `ai/mcp/server/<name>/` (`file-system`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `memory-core`, `neural-link`) with its own `openapi.yaml` (`info.version: 1.0.0`) and `mcp-server.mjs` entrypoint. Co-locating `server.json` inside each `ai/mcp/server/<name>/` directory keeps the registry metadata beside `openapi.yaml` and allows `mcp-publisher` to publish each server directly from its `repository.subfolder`.

## The Fix

1. Add `ai/mcp/server/<name>/server.json` for all 6 MCP servers (`file-system`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `memory-core`, `neural-link`) conforming strictly to `https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json`.
2. Add a Playwright unit test `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` validating all 6 `server.json` files against the `2025-12-11` MCP Registry schema constraints (allowed keys, `io.github.neomjs/<dir>` naming, `description` length `1..100`, `title` length `1..100`, `version` parity with sibling `openapi.yaml`, and `repository.subfolder` existence on disk).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}/server.json` | `https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json` + sibling `openapi.yaml` | Provides official MCP Registry `server.json` manifests under `io.github.neomjs/<server>` | Local Fleet Manager and workspace `.mcp.json` launch configs continue to work without registry lookup | `ai/mcp/server/*/server.json` | `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` |

Decision Record impact: `none`

## Acceptance Criteria

- [ ] All 6 MCP server directories (`ai/mcp/server/{file-system,github-workflow,gitlab-workflow,knowledge-base,memory-core,neural-link}`) contain a `server.json` file with `$schema: "https://static.modelcontextprotocol.io/schemas/2025-12-11/server.schema.json"` and `name: "io.github.neomjs/<server>"`.
- [ ] Every `server.json` satisfies the `ServerDetail` and `Repository` schema bounds (`description` `1..100` chars, `title` `1..100` chars, `version` matching sibling `openapi.yaml`'s `info.version`, `repository.id: "1344062228"`, `repository.source: "github"`, `repository.url: "https://github.com/neomjs/neo-agent-brain"`, and `repository.subfolder: "ai/mcp/server/<server>"`).
- [ ] Unit test `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` verifies all 6 manifests and passes via `npm run test-unit -- test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs`.

## Post-Merge Validation

- [ ] Gate the Oct 31 `mcp-publisher` publish click on adding `packages` entries from the public one-step install npm package (`@neo-opus-grace`, `pullrequestreview-5641238244`) so registry clients have a runnable npm package identifier (`registryType: "npm"`) rather than metadata-only entries pointing at private `neo-agent-brain`.
- [ ] Publish the 6 `ai/mcp/server/*/server.json` manifests to `registry.modelcontextprotocol.io` via `mcp-publisher` on the Oct 31 directory release window once `packages` entries are populated.

## Out of Scope

- Running `mcp-publisher publish` against `registry.modelcontextprotocol.io` prior to the Oct 31 directory release window.
- Modifying runtime MCP server transport or tool definitions.

## Related

- Sibling: `neomjs/neo#20101` (`#20102`), `neomjs/pages#53` (`#54`), `neomjs/middleware-v2#41` (`#42`)

Origin Session ID: `2d131263-cb25-4f19-8a19-98ef961cc86b`
Retrieval Hint: `"registry.modelcontextprotocol.io server.json MCP Registry neomjs/neo-agent-brain"`

## Timeline

- 2026-10-25T02:12:35Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-25T02:12:35Z @neo-gemini-julia added the `enhancement` label
- 2026-10-25T02:12:35Z @neo-gemini-julia added the `ai` label
- 2026-10-25T02:12:35Z @neo-gemini-julia added the `model-experience` label
- 2026-10-25T02:12:36Z @neo-gemini-julia added the `agent-os` label
- 2026-10-25T02:15:54Z @neo-gemini-julia cross-referenced by PR #1266
- 2026-10-25T03:09:15Z @tobiu closed this issue
- 2026-10-25T03:09:15Z @tobiu referenced in commit `d50ed09` - "feat(mcp): add MCP Registry server.json manifests for the six MCP servers (#1265) (#1266)"
### @neo-opus-grace - 2026-10-25T23:58:10Z

This ticket's post-merge publish gate now has an owner. The `neural-link` manifest's npm `packages` entry, and the package it points at, are neomjs/neo-agent-brain#1282, under neomjs/neo#20114 (graduated from neomjs/neo D#20113).

The other five manifests (`file-system`, `github-workflow`, `gitlab-workflow`, `knowledge-base`, `memory-core`) stay unpublished until a client can run them, as D#20113 decided. So the Oct 31 publish covers `neural-link` alone, once its package is live.

🖖 Grace (Opus 5.5, Claude Code) · session 5597537c-fa0a-4b88-8847-edddb7492a11


