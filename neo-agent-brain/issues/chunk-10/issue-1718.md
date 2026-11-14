---
id: 1718
title: Generalize packMcpServer for neo-memory-core and cross-harness walk
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - agent-os
  - github_actions
assignees: []
createdAt: '2026-11-14T17:54:27Z'
updatedAt: '2026-11-14T17:54:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1718'
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
  - '[ ] 1717 Add rostered-owner standalone WAL promotion helper and RLS spec'
  - '[ ] 1716 Add standalone memory-core MCP server, SQLite FTS5 index, and ADR'
blocking:
  - '[ ] 1719 Publish neo-memory-core to npm and dispatch MCP Registry'
---
# Generalize packMcpServer for neo-memory-core and cross-harness walk

## Context

Graduated as **Leaf 3** of Parent Epic [`neomjs/neo-agent-brain#1715`](https://github.com/neomjs/neo-agent-brain/issues/1715) from [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941) (`OQ4`, `OQ5`, and `STEP_BACK` Points 5, 6, 8 at `DC_kwDODSospM4BK1En`).

Once Leaf 1 (standalone server + `node:sqlite` index) and Leaf 2 (`neo-memory-core promote`) land, we need a single shared MCP npm package assembler (generalizing `ai/scripts/setup/packNeuralLink.mjs` rather than copying it), updated MCP Registry metadata in `ai/mcp/server/memory-core/server.json` + `.github/workflows/publish-mcp-registry.yml`, empirical Node `22.22+` / `24.x` / `25.x` cold stdio validation (`F6`), and an automated clean-HOME **cross-harness two-session cold walk** (`F1`) proving zero heavy dependencies and `<2s` cross-harness recall before npm publication.

- `Live latest-open + closed sweep:` checked latest 20 open issues (`155` open — red backlog band) + closed search (`packNeuralLink server.json McpRegistryManifests`, `Out of Scope` / `Avoided Traps` read on `#1282` and `#1265`) on `neomjs/neo-agent-brain` at `2026-11-14T17:50:16Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 `all`-status messages at `2026-11-14T17:50:16Z`; no competing claim found.
- `MC sweep:` `query_raw_memories` (`packNeuralLink McpRegistryManifests publish-mcp-registry`), surfaced `#1282` / `#1265` precedents.
- `Own-assignment sweep:` checked open `@me` issues (`#1548` and `#1230` — both orthogonal).
- `Structure-map gate:` executed `npm run ai:structure-map -- --files --loc` (`exit 0`); sibling precedent verified at `ai/scripts/setup/packNeuralLink.mjs` (`251` lines).
- Ownership: maintainer lane (`good first issue` / `help wanted` omitted because this leaf modifies `.github/workflows/publish-mcp-registry.yml` release gates and shared packaging infrastructure; `blocked_by` Leaf 1 and Leaf 2).

---

## The Problem

`ai/scripts/setup/packNeuralLink.mjs` (`:1-251`) hardcodes `neo-neural-link` entrypoints, assets, `bin`, `mcpName`, and generated `README.md`. Duplicating a 250-line script for `neo-memory-core` would violate `AGENTS.md §self_evolving_systems` Accretion Defense (`STEP_BACK` #8). Meanwhile, `ai/mcp/server/memory-core/server.json` (`io.github.neomjs/memory-core`) lacks a `packages` block, `McpRegistryManifests.spec.mjs` (`:55-61`) only asserts `packages` on `neural-link`, and `.github/workflows/publish-mcp-registry.yml` (`:13-17`) only exposes `neural-link` in `workflow_dispatch`.

---

## The Architectural Reality

- Design authority: n/a — generalizes `packNeuralLink.mjs` to assemble `neo-memory-core` alongside `neo-neural-link` per Discussion #20941 `STEP_BACK` #8 without calling existing `neural-link` packaging wrong.
- `ai/scripts/setup/packNeuralLink.mjs:33-112` already walks transitive relative ES module imports via `collectClosure(entries)` and resolves external runtime `dependencies` against root `package.json` + `node_modules/neo.mjs/package.json`.
- `test/playwright/unit/ai/scripts/setup/packNeuralLink.spec.mjs:13` enforces `FORBIDDEN_RUNTIME_DEPS` so heavy Agent OS packages never leak into cold `npx` installs.
- `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` validates all `ai/mcp/server/*/server.json` manifests against the official MCP Registry schema and cross-checks `packages[0]` against the packed `package.json`.
- Structural Pre-Flight (`Stage 1` fast-path): `ai/scripts/setup/packMcpServer.mjs` matches `ai/scripts/setup/packNeuralLink.mjs` in `ai/scripts/setup/`.

---

## The Fix

1. **Generalize `ai/scripts/setup/packMcpServer.mjs` & Keep `packNeuralLink.mjs` Thin:**
   - Extract shared `collectClosure`, dependency pruning (`node:*` built-in `node:sqlite` excluded from npm `dependencies`), asset copying, `package.json` / `README.md` generation, and `npm pack` tarball assembly into `ai/scripts/setup/packMcpServer.mjs` with declarative target definitions for `neural-link` (`neo-neural-link`) and `memory-core` (`neo-memory-core`, `mcpName: 'io.github.neomjs/memory-core'`, `bin: {'neo-memory-core': 'ai/mcp/server/memory-core/standalone.mjs'}`, assets `ai/mcp/server/memory-core/openapi.yaml` + `ai/mcp/server/memory-core/server.json`).
   - Refactor `ai/scripts/setup/packNeuralLink.mjs` into a thin compatibility entrypoint delegating to `packMcpServer.mjs` (`--target neural-link`) and add `npm run ai:pack-memory-core` (`packMcpServer.mjs --target memory-core`).
   - Measure cold stdio MCP framing (`initialize`, `tools/list`, `add_memory`, `query_raw_memories`) on available Node lines (`22.22+` if installed/probed, `24.x`, `25.x` — checking `stderr` for `ExperimentalWarning` or framing corruption per `F6`) and pin packed `engines.node` to the empirically verified clean floor (`>=22.5.0` if stdio-clean, else `>=24.0.0`).
2. **Update MCP Registry Manifest & Workflow (`server.json`, `McpRegistryManifests.spec.mjs`, `publish-mcp-registry.yml`):**
   - Add `packages` (`registryType: 'npm'`, `identifier: 'neo-memory-core'`, `transport: {type: 'stdio'}`) to `ai/mcp/server/memory-core/server.json`.
   - Extend `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` to cross-check `memory-core/server.json` against `packMcpServer` (`memory-core`).
   - Add `'memory-core'` to `.github/workflows/publish-mcp-registry.yml` `inputs.server.options` (`neural-link`, `memory-core`, `all` — keeping unpublished servers excluded until their standalone packages exist).
3. **Add Closure & Clean-HOME Cross-Harness Two-Session Spec (`packMcpServer.spec.mjs`):**
   - Assert `chromadb`, `better-sqlite3`, and `@chroma-core/default-embed` are absent from `neo-memory-core`'s `package.json` `dependencies` and packed file closure (`STEP_BACK` #5).
   - Unpack the generated `neo-memory-core-<ver>.tgz` into an isolated temp directory with a clean `HOME` (no checkout, no Docker): spawn **Session 1** simulating Harness A (`clientInfo: {name: 'claude-code'}`, `cwd` at repo root) calling `add_memory`, terminate it, then spawn **Session 2** simulating Harness B (`clientInfo: {name: 'codex-cli'}`, `cwd` inside a nested subdirectory of the same git repo) and assert `query_recent_turns`, `get_session_memories`, and `query_raw_memories` recall Session 1's turn in `<2000ms` with zero writes inside the unpacked package root (`F1`).

---

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `packMcpServer.mjs` & `packNeuralLink.mjs` | `Discussion #20941` `STEP_BACK` #5, #8 | Single shared packer assembling `neo-neural-link` and `neo-memory-core` with forbidden-dep guard (`chromadb`, `better-sqlite3`, `@chroma-core/default-embed` absent) | Fails build if closure imports forbidden packages | Generated package `README.md` | `packMcpServer.spec.mjs` + `packNeuralLink.spec.mjs` |
| `ai/mcp/server/memory-core/server.json` & `.github/workflows/publish-mcp-registry.yml` | MCP Registry `2025-12-11` schema, `McpRegistryManifests.spec.mjs` | Declares `neo-memory-core` stdio npm package for `io.github.neomjs/memory-core` and adds `memory-core` workflow option | Workflow verifies live npm tarball before publishing to Registry | `memory-core/server.json` | `McpRegistryManifests.spec.mjs` |

- **Decision Record:** `Optional: neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`
- **Decision Record impact:** `aligned-with neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`

---

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: standalone-memory-core-contract -->
- `ai/scripts/setup/packMcpServer.mjs` — shared MCP npm package assembler for `neo-neural-link` and `neo-memory-core`
- `ai/scripts/setup/packNeuralLink.mjs` — thin wrapper delegating to `packMcpServer.mjs` so existing callers stay unbroken
- `ai/mcp/server/memory-core/server.json` — MCP Registry manifest (`io.github.neomjs/memory-core`) with `neo-memory-core` npm stdio package entry
- `test/playwright/unit/ai/mcp/server/McpRegistryManifests.spec.mjs` — cross-checks `memory-core/server.json` against packed `neo-memory-core` `package.json`
- `.github/workflows/publish-mcp-registry.yml` — exposes `memory-core` in `workflow_dispatch` server choices alongside `neural-link`
<!-- /binding-inventory -->

---

## Acceptance Criteria

- [ ] **AC-1:** `packMcpServer.mjs` builds both `neo-neural-link` (`packNeuralLink.spec.mjs` green without regressions) and `neo-memory-core` (`npm run ai:pack-memory-core`), asserting `chromadb`, `better-sqlite3`, and `@chroma-core/default-embed` are absent from both `dependencies` and packed files, and pins `engines.node` to the empirically measured clean stdio floor (`F6` receipt recorded in PR `## Test Evidence`).
- [ ] **AC-2:** `ai/mcp/server/memory-core/server.json`, `McpRegistryManifests.spec.mjs`, and `.github/workflows/publish-mcp-registry.yml` include `neo-memory-core` / `memory-core` and pass `McpRegistryManifests.spec.mjs`.
- [ ] **AC-3:** `packMcpServer.spec.mjs` unpacks the packed `neo-memory-core` tarball into a clean `HOME` temp directory and passes the two-process cross-harness walk (Session 1 `claude-code` at repo root -> `add_memory`; Session 2 `codex-cli` in a nested subdir -> `query_recent_turns` + `get_session_memories` + `query_raw_memories` recall in `<2000ms`, zero writes inside the unpacked package root).

---

## Out of Scope

- Running live `npm publish` or dispatching `publish-mcp-registry.yml` against production registries (owned by Leaf 6 after Leaf 3 merges to `dev`).

---

## Avoided Traps

- **Copy-pasting `packNeuralLink.mjs` into `packMemoryCore.mjs`:** Doubles maintenance surface for closure walking and tarball verification (`STEP_BACK` #8).
- **Enabling `all` in `publish-mcp-registry.yml` across unpublished servers:** Keep `publish-mcp-registry.yml` restricted to servers with live standalone npm packages (`neural-link` and `memory-core`).

---

## Related

- Parent Epic: #1715
- Source Discussion: [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941)

Origin Session ID: `2f084437-cc03-4b7b-aace-a27bff4aa3f9`
Retrieval Hint: `"packMcpServer" "packNeuralLink" "McpRegistryManifests" "publish-mcp-registry" "io.github.neomjs/memory-core"`

## Timeline

- 2026-11-14T17:54:29Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T17:54:29Z @neo-gemini-julia added the `ai` label
- 2026-11-14T17:54:29Z @neo-gemini-julia added the `build` label
- 2026-11-14T17:54:29Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T17:54:29Z @neo-gemini-julia added the `github_actions` label
- 2026-11-14T17:55:13Z @neo-gemini-julia added parent issue #1715
- 2026-11-14T17:55:22Z @neo-gemini-julia marked this issue as being blocked by #1716
- 2026-11-14T17:55:25Z @neo-gemini-julia marked this issue as being blocked by #1717
- 2026-11-14T17:55:28Z @neo-gemini-julia marked this issue as blocking #1719

