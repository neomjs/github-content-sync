---
id: 1719
title: Publish neo-memory-core to npm and dispatch MCP Registry
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
  - github_actions
assignees: []
createdAt: '2026-11-14T17:54:34Z'
updatedAt: '2026-11-14T17:54:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1719'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 1715
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1718 Generalize packMcpServer for neo-memory-core and cross-harness walk'
blocking: []
---
# Publish neo-memory-core to npm and dispatch MCP Registry

## Context

Graduated as **Leaf 6** (Final Release & Registry Gate) of Parent Epic [`neomjs/neo-agent-brain#1715`](https://github.com/neomjs/neo-agent-brain/issues/1715) from [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941) (`OQ5` + `§6.5`).

Once Leaves 1–3 merge to `neomjs/neo-agent-brain` `dev`, this leaf executes and records the live npm publication of `neo-memory-core`, dispatches `.github/workflows/publish-mcp-registry.yml` (`server: memory-core`) to activate `io.github.neomjs/memory-core` in the official MCP Registry, and verifies the published `npx -y neo-memory-core` package via a live clean-HOME cross-harness two-session smoke walk (`F1`).

- `Live latest-open + closed sweep:` checked latest 20 open issues (`155` open — red backlog band) + closed search (`publish-mcp-registry neo-neural-link`, `Out of Scope` read on `#1266` and `#1282`) on `neomjs/neo-agent-brain` at `2026-11-14T17:50:16Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 `all`-status messages at `2026-11-14T17:50:16Z`; no competing claim found.
- `MC sweep:` `query_raw_memories` (`publish-mcp-registry neo-neural-link io.github.neomjs`), surfaced `#1266` / `#1282` release receipts.
- `Own-assignment sweep:` checked open `@me` issues (`#1548` and `#1230` — both orthogonal).
- `Structure-map gate:` executed `npm run ai:structure-map -- --files --loc` (`exit 0`); release workflow verified at `.github/workflows/publish-mcp-registry.yml`.
- Ownership: maintainer/operator release lane (`good first issue` / `help wanted` omitted because npm publish + GitHub OIDC MCP Registry workflow dispatch require maintainer/operator release credentials; `blocked_by` Leaf 3).

---

## The Problem

`npm view neo-memory-core` currently returns `E404` and `io.github.neomjs/memory-core` is unpublished in the official MCP Registry (`https://registry.modelcontextprotocol.io/v0.1/servers/io.github.neomjs%2Fmemory-core/versions/latest`). Because `.github/workflows/publish-mcp-registry.yml` verifies the published npm tarball (`mcpName === "io.github.neomjs/memory-core"`) before calling `mcp-publisher publish`, npm publication must precede Registry dispatch and be verified from the live registry via `npx -y neo-memory-core`.

---

## The Architectural Reality

- Design authority: n/a — executes the npm and MCP Registry release for `neo-memory-core` graduated in Discussion #20941 following the `neo-neural-link` (`#1266` / `#1282`) release sequence.
- `.github/workflows/publish-mcp-registry.yml` authenticates via GitHub OIDC (`mcp-publisher login github-oidc`), validates `npm view neo-memory-core dist.tarball` `package.json.mcpName`, and publishes `ai/mcp/server/memory-core/server.json` to `registry.modelcontextprotocol.io`.

---

## The Fix

1. **Pack & Publish `neo-memory-core` to npm (Operator/Maintainer Gate):**
   - Run `npm run ai:pack-memory-core`, verify tarball contents and `mcpName: "io.github.neomjs/memory-core"`, publish `neo-memory-core` to npm (coordinating with `@tobiu` for npm OTP/token if required), and confirm `npm view neo-memory-core version mcpName` resolves live.
2. **Dispatch `publish-mcp-registry.yml` (`server=memory-core`):**
   - Trigger `.github/workflows/publish-mcp-registry.yml` on `neomjs/neo-agent-brain` for `memory-core` and verify `https://registry.modelcontextprotocol.io/v0.1/servers/io.github.neomjs%2Fmemory-core/versions/latest` returns `status: "active"`.
3. **Live Registry Cross-Harness Cold Walk Receipt (`F1`):**
   - From a clean temporary `HOME` and git repository outside `neomjs/*`, run `npx -y neo-memory-core` from npm across two distinct client sessions (`claude-code` -> `add_memory`, `codex-cli` -> `query_recent_turns` + `query_raw_memories`) and record the live version, latency (`<2s`), and Registry URL receipt on the ticket / release doc before closing.

---

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-memory-core` (npm) | `Discussion #20941` `§4.1`, `packMcpServer.mjs` | Published npm package with `bin.neo-memory-core` and `mcpName: "io.github.neomjs/memory-core"` | Hold Registry dispatch until `npm view neo-memory-core` resolves | Generated package `README.md` | `npm view neo-memory-core` receipt |
| `io.github.neomjs/memory-core` (MCP Registry) | `ai/mcp/server/memory-core/server.json`, `.github/workflows/publish-mcp-registry.yml` | Active MCP Registry entry pointing to `neo-memory-core` on npm | Re-run idempotent `publish-mcp-registry.yml` workflow | `memory-core/server.json` | Live Registry JSON + `npx -y neo-memory-core` two-session receipt |

- **Decision Record:** `Optional: neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`
- **Decision Record impact:** `none`

---

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: standalone-memory-core-contract -->
- `neo-memory-core` — published standalone npm MCP package (`npx -y neo-memory-core`)
- `io.github.neomjs/memory-core` — active official MCP Registry entry published via `publish-mcp-registry.yml`
<!-- /binding-inventory -->

---

## Acceptance Criteria

- [ ] **AC-1:** `neo-memory-core` is published on npm with `mcpName: "io.github.neomjs/memory-core"`, verified via `npm view neo-memory-core version mcpName dist.tarball`.
- [ ] **AC-2:** `.github/workflows/publish-mcp-registry.yml` (`server=memory-core`) succeeds and `https://registry.modelcontextprotocol.io/v0.1/servers/io.github.neomjs%2Fmemory-core/versions/latest` reports `io.github.neomjs/memory-core` active.
- [ ] **AC-3:** Live `npx -y neo-memory-core` fetched from npm passes the clean-HOME two-session cross-harness recall smoke test (`<2s` Session 2 recall, zero `~/.npm/_npx` plane writes, zero git-tree writes) with receipt recorded.

---

## Out of Scope

- Publishing other Agent OS MCP servers (`github-workflow`, `knowledge-base`) to npm or the MCP Registry.

---

## Avoided Traps

- **Dispatching `publish-mcp-registry.yml` before npm propagation:** The MCP Registry verifies live npm tarball ownership (`mcpName`) and rejects unpublished packages.

---

## Related

- Parent Epic: #1715
- Source Discussion: [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941)
- Sibling precedent: [`neomjs/neo-agent-brain#1266`](https://github.com/neomjs/neo-agent-brain/issues/1266) (`io.github.neomjs/neural-link`)

Origin Session ID: `2f084437-cc03-4b7b-aace-a27bff4aa3f9`
Retrieval Hint: `"neo-memory-core" "io.github.neomjs/memory-core" "publish-mcp-registry"`

## Timeline

- 2026-11-14T17:54:36Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T17:54:36Z @neo-gemini-julia added the `ai` label
- 2026-11-14T17:54:36Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T17:54:36Z @neo-gemini-julia added the `github_actions` label
- 2026-11-14T17:55:16Z @neo-gemini-julia added parent issue #1715
- 2026-11-14T17:55:28Z @neo-gemini-julia marked this issue as being blocked by #1718

