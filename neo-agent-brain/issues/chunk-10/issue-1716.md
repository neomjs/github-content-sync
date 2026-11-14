---
id: 1716
title: 'Add standalone memory-core MCP server, SQLite FTS5 index, and ADR'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-14T17:54:21Z'
updatedAt: '2026-11-14T19:42:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1716'
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
blockedBy: []
blocking:
  - '[ ] 1718 Generalize packMcpServer for neo-memory-core and cross-harness walk'
  - '[ ] 1717 Add rostered-owner standalone WAL promotion helper and RLS spec'
---
# Add standalone memory-core MCP server, SQLite FTS5 index, and ADR

## Context

Graduated as **Leaf 1** of Parent Epic [`neomjs/neo-agent-brain#1715`](https://github.com/neomjs/neo-agent-brain/issues/1715) from [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941) (`M1' + W2 + E1`, quorum met via `gemini` `AUTHOR_SIGNAL` at `DC_kwDODSospM4BK1G5` + `claude` `GRADUATION_APPROVED` at `DC_kwDODSospM4BK1Hk`).

External coding agents need cross-harness persistent episodic memory (`npx -y neo-memory-core`) with zero Docker containers, zero LLM/embedding daemons, and zero native C++ addon prerequisites (`better-sqlite3`). This leaf delivers the standalone MCP launcher, server, lean tool binder, 3-member `ConfigProvider`, deterministic workspace resolver, `node:sqlite` WAL/FTS5 index with per-read byte-watermark catch-up, contiguous 6-tool `openapi.yaml` exact profile, and the required `ADR-0019 §10.7` profile row.

- `Live latest-open + closed sweep:` checked latest 20 open issues (`155` open — red backlog band) + closed search (`standalone memory-core sqlite FTS5`, `Out of Scope` / `Avoided Traps` read on `#1282`) on `neomjs/neo-agent-brain` at `2026-11-14T17:50:16Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 `all`-status messages at `2026-11-14T17:50:16Z`; no competing claim found.
- `MC sweep:` `query_raw_memories` (`standalone memory-core sqlite FTS5 openapi profile`), no prior implementation ticket found.
- `Own-assignment sweep:` checked open `@me` issues (`#1548` and `#1230` — both orthogonal).
- `Structure-map gate:` executed `npm run ai:structure-map -- --files --loc` (`exit 0`); sibling precedents verified at `ai/mcp/server/neural-link/standalone.mjs`, `ai/mcp/server/memory-core/{Server,toolService,configBase}.mjs`, and `ai/services/memory-core/helpers/memoryWalStore.mjs`.
- Ownership: self-assigned to `@neo-gemini-julia` for immediate execution (`good first issue` / `help wanted` omitted because this foundational runtime leaf is actively owned in-lane).

---

## The Problem

`ai/mcp/server/memory-core/Server.mjs` (`Server.mjs:91`) returns `isPlaneMember() === true` over `BaseServer#collectMemberEntries` (`ai/mcp/server/BaseServer.mjs:725-738`), which concatenates `11` server paths (`ai/mcp/server/memory-core/configBase.mjs:1058-1073`) with all `12` `TIER1_PLANE_MEMBER_PATHS` (`ai/configBase.mjs`), failing `assertPlaneMemberCoherence` outside a full 23-member Agent OS deployment. Furthermore, `ai/mcp/server/memory-core/toolService.mjs` (`toolService.mjs:8-24`) statically imports `GraphService` (`better-sqlite3`), `MemoryService` (`chromadb` + `@chroma-core/default-embed`), and `SessionService`, pulling heavy dependencies and exposing 56 full-plane tools instead of a zero-daemon standalone surface.

---

## The Architectural Reality

- Design authority: n/a — introduces the new standalone `npx` runtime profile and exact tool projection graduated in Discussion #20941 without calling existing full-plane `Server.mjs` behavior wrong.
- `ai/mcp/ToolService.mjs:753-769, 846-853` already compiles `x-neo-exact-tool-profiles.profiles.<mode>` fail-closed when `toolProjectionMode` is set (`'standalone'`) and serves `get_mcp_tool_handbook` when `get_mcp_tool_handbook: {}` is listed in `exactProfile.tools`.
- `ai/services/memory-core/helpers/memoryWalStore.mjs:122-185` (`appendWalMemory` + `walAppendLock.mjs`) appends canonical `wal-YYYY-MM-DD.jsonl` records atomically across processes while spreading top-level record fields (`workspace`, `workspaceSource`) and keeping `.embedded.jsonl` (`:131`) and `.graph.jsonl` (`:140`) markers separate.
- Under `ADR-0019` (`learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` `§5, §10.1, §10.3–§10.7`), a standalone server owning durable state declares a `ConfigProvider` with `getParent() { return null; }`, opaque `plane.id`, absolute `plane.dataRoot` (`~/.neo-ai/memory-core`), pinned `'stdio'` `transport` (no `NEO_TRANSPORT` env binding), and explicit `PLANE_MEMBER_PATHS` (`memoryWal.dirProd`, `standaloneIndexPath`, `logPath` `<root>/logs` directory) declared from one pure `standalonePlacement` helper, enforced on both launcher and `--serve` entry paths, and checked by `assertPlaneCoherence` + `assertPlaneMemberCoherence` (`collectPlaneMembers` called directly over `STANDALONE_PLANE_MEMBER_PATHS`).
- Structural Pre-Flight (`Stage 1` fast-path verified for all 6 new `.mjs` files): `standalone.mjs` matches `ai/mcp/server/neural-link/standalone.mjs`; `standaloneServer.mjs`, `standaloneToolService.mjs`, and `standaloneConfigBase.mjs` match `Server.mjs`, `toolService.mjs`, and `configBase.mjs` in `ai/mcp/server/memory-core/`; `standaloneWorkspaceKey.mjs` and `standaloneSqliteIndex.mjs` match `memoryWalStore.mjs` in `ai/services/memory-core/helpers/`.

---

## The Fix

1. **Workspace Resolver (`ai/services/memory-core/helpers/standaloneWorkspaceKey.mjs`):**
   - Implement `resolveStandaloneWorkspaceKey({explicitWorkspace, cwd, execGit})` following `explicit-slug` (non-path `--workspace <slug>` / `NEO_MC_WORKSPACE`) -> `git-remote` (normalized `git remote get-url origin` as `<lowercase-host>/<owner>/<repo>`, credentials and `.git` stripped) -> `git-toplevel` (`realpathSync(git rev-parse --show-toplevel)`) -> `cwd` (`realpathSync(cwd)`), returning `{workspace, workspaceSource}`.
2. **Derived `node:sqlite` WAL/FTS5 Index (`ai/services/memory-core/helpers/standaloneSqliteIndex.mjs`):**
   - Built on `node:sqlite` (`DatabaseSync`), `PRAGMA journal_mode=WAL`, `PRAGMA busy_timeout=5000`, FTS5 virtual table (`memories_fts` with `bm25()`), and `index_meta(key TEXT PRIMARY KEY, value TEXT)` storing `(latestSegmentKey, latestSegmentBytes)` watermark state; all DML uses bound `?` parameters (`SQLite 3.51` `DQS_DML=0` safe).
   - Provide boot rebuild (when `index.sqlite` is missing or empty while `wal-*.jsonl` segments exist), incremental byte-watermark catch-up (`syncFromWalWatermark()` invoked before every read/search), `upsertMemory(record)`, `queryRecentTurns(...)` (`detail: 'summary'` returning `summaryFallback: true` + bounded deterministic excerpt; workspace-scoped by default, `workspace: '*'` opt-in), `queryRawMemories(...)` (returning `retrievalMode: 'lexical-fts5'`, raw negative `bm25Score`, and relevance-rising `lexicalRankScore: |bm25| / (1 + |bm25|)`, omitting `distance` and `compositeScore`), `getSessionMemories(...)`, and `getStorageStats()` (`{walBytes, indexBytes, memoryCount}`).
3. **Standalone Config, Tool Binder, Server & Launcher (`ai/mcp/server/memory-core/{standaloneConfigBase,standaloneToolService,standaloneServer,standalone}.mjs`):**
   - `standaloneConfigBase.mjs`: `ConfigProvider` subclass (`getParent() { return null; }`) declaring leaf defaults from `standalonePlacement` (`ADR-0019 §10.1`) for `plane.{id, dataRoot}`, `memoryWal.dirProd`, `standaloneIndexPath`, `logPath` (`<root>/logs` directory), `loggerRetention`, `logger`, pinned `'stdio'` `transport` (no `NEO_TRANSPORT` env binding), `workspace`, and `sessionId` with `STANDALONE_PLANE_MEMBER_PATHS` (`3` members, complete under `derivePlaneMemberPaths`).
   - `standaloneToolService.mjs`: instantiates `ToolService` directly over `ai/mcp/server/memory-core/openapi.yaml` (`toolProjectionMode: 'standalone'`) with zero imports of `GraphService`, `MemoryService`, or `SessionService`, binding the 6 standalone tools (`add_memory` writes unmarked `wal-YYYY-MM-DD.jsonl` via `appendWalMemory` — writing neither `.embedded.jsonl` nor `.graph.jsonl`, keeping `metadata.agent` for display while omitting unverified `agentIdentity` until `#1717`'s rostered `--owner` stamps it — and upserts `index.sqlite`; honors explicit `sessionId` with per-process UUID fallback).
   - `standaloneServer.mjs`: `BaseServer` subclass (`isPlaneMember() { return true; }`, overriding `getPlaneMembers()` via `collectPlaneMembers` over `STANDALONE_PLANE_MEMBER_PATHS`), failing closed on non-`stdio` transport overlays and plane incoherence before creating files, wiring shared `createLogger` (`ai/mcp/server/shared/logger.mjs`), setting MCP `initialize` `instructions` to guide connected harnesses to call `query_recent_turns` at session boot and `add_memory` at task completion, and returning `{plane: {id, dataRoot}, workspace, workspaceSource, storage: {walBytes, indexBytes, memoryCount, retentionPolicy: 'none-v1'}}` on `healthcheck`.
   - `standalone.mjs`: CLI launcher (`#!/usr/bin/env node`) parsing `--workspace`, `--data-root`, `--session-id`, `--config`, `--serve`, `--debug`, and `--help`/`--version`, applying `standalonePlacement` (`NEO_PLANE_ID`, `NEO_PLANE_DATA_ROOT`, `NEO_MEMORY_WAL_DIR`, `NEO_MC_STANDALONE_INDEX_PATH`, `NEO_MC_LOG_PATH`) on both the supervised child-launcher path and the direct `--serve` entry path so standalone placement wins over inherited foreign plane env on every entry path.
4. **OpenAPI Profile & `ADR-0019 §10.7` Amendment:**
   - Add one contiguous `x-neo-exact-tool-profiles.profiles.standalone` block to `ai/mcp/server/memory-core/openapi.yaml` for the 6 tools (noting `lexical-fts5` / `summaryFallback: true` behavior in profile descriptions and optional `workspace` filter parameter where needed).
   - Add one `Standalone Memory Core package (npx)` row to `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` `§10.7`, verified against `ADR-0019 §5, §10.1, §10.3–§10.6, §10.9`.

---

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `standalone.mjs` / `standaloneConfigBase.mjs` / `standaloneServer.mjs` | `ADR-0019 §10.1, §10.3–§10.7`, `neural-link/standalone.mjs`, `shared/logger.mjs` | Applies `standalonePlacement` (`~/.neo-ai/memory-core`: `wal/`, `index.sqlite`, `logs/`) with opaque `plane.id` and pinned `'stdio'` transport on both launcher and `--serve` entry paths (`--workspace`, `--data-root`, `--session-id`, `--config`, `--debug`, `--help`/`--version`; `promote` deferred to `#1717`); passes `assertPlaneCoherence` + `assertPlaneMemberCoherence` over 3 members | `--data-root` / `NEO_PLANE_DATA_ROOT` explicit relocation with all 3 members placed; non-`stdio` overlays fail closed | `0019-aiconfig-reactive-provider-ssot.md` `§10.7` | Playwright unit spec |
| `standaloneWorkspaceKey.mjs` | `Discussion #20941` `STEP_BACK` #3 | Resolves `explicit-slug -> git-remote -> git-toplevel -> cwd`; reports `{workspace, workspaceSource}` on `healthcheck` | `realpath(process.cwd())` outside git | `openapi.yaml` / `0019` | Playwright unit spec |
| `x-neo-exact-tool-profiles.profiles.standalone` (`openapi.yaml` + `standaloneToolService.mjs`) | `ai/mcp/ToolService.mjs:753-853`, `Discussion #20941` `OQ2` | Exposes exact 6 tools (`healthcheck`, `get_mcp_tool_handbook`, `add_memory`, `query_recent_turns`, `query_raw_memories`, `get_session_memories`); `summaryFallback: true` excerpt; `lexical-fts5` + `bm25Score` + `lexicalRankScore` (`|bm25| / (1 + |bm25|)`); `add_memory` keeps `metadata.agent` without stamping unverified `agentIdentity` | Fail-closed rejection on all other 50 Memory Core tools | `openapi.yaml` | Playwright unit spec |
| `standaloneSqliteIndex.mjs` + `appendWalMemory` | `memoryWalStore.mjs:122-185`, `Discussion #20941` `OQ1` | Writes unmarked `wal-YYYY-MM-DD.jsonl` (neither `.embedded.jsonl` nor `.graph.jsonl`), indexes into `node:sqlite` WAL/FTS5, catches up via `stat.size` byte watermark on reads, rebuilds if `index.sqlite` deleted | Automatic full WAL replay when `index.sqlite` is absent | `0019-aiconfig-reactive-provider-ssot.md` | Playwright unit spec |

- **Decision Record:** `Required: neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`
- **Decision Record impact:** `amends neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`

---

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: standalone-memory-core-contract -->
- `ai/mcp/server/memory-core/standalone.mjs` — CLI stdio launcher placing `~/.neo-ai/memory-core` and 3 plane-member env slots on both launcher and `--serve` entry paths
- `ai/mcp/server/memory-core/standaloneServer.mjs` — `BaseServer` subclass owning 3-member plane coherence, pinned `'stdio'` transport guard, shared `createLogger`, MCP `initialize` `instructions`, and `healthcheck` gauges
- `ai/mcp/server/memory-core/standaloneToolService.mjs` — lean `ToolService` binder (`toolProjectionMode: 'standalone'`) free of `GraphService`/`MemoryService`/`SessionService` imports
- `ai/mcp/server/memory-core/standaloneConfigBase.mjs` — standalone `ConfigProvider` (`getParent() { return null; }`) declaring leaf defaults from `standalonePlacement` (`plane.{id,dataRoot}` + 3 `PLANE_MEMBER_PATHS`)
- `ai/services/memory-core/helpers/standaloneWorkspaceKey.mjs` — deterministic `explicit-slug -> git-remote -> git-toplevel -> cwd` workspace resolver
- `ai/services/memory-core/helpers/standaloneSqliteIndex.mjs` — derived `node:sqlite` (`DatabaseSync` WAL + FTS5 `bm25()`) index with byte-watermark catch-up and full WAL rebuild
- `ai/mcp/server/memory-core/openapi.yaml` — hosts the contiguous `standalone` exact tool profile schema
- `x-neo-exact-tool-profiles.profiles.standalone` — fail-closed 6-tool projection profile
- `healthcheck` — reports `{plane: {id, dataRoot}, workspace, workspaceSource, storage: {walBytes, indexBytes, memoryCount, retentionPolicy: 'none-v1'}}`
- `get_mcp_tool_handbook` — standalone tool handbook introspection
- `add_memory` — appends unmarked `wal-YYYY-MM-DD.jsonl` with `{workspace, workspaceSource}` (omitting unverified `agentIdentity`) and upserts `index.sqlite`
- `query_recent_turns` — workspace-scoped reverse-chronological recall (`detail: 'summary'` sets `summaryFallback: true` + deterministic excerpt)
- `query_raw_memories` — workspace-scoped `node:sqlite` FTS5 BM25 search (`retrievalMode: 'lexical-fts5'`, `bm25Score`, `lexicalRankScore: |bm25| / (1 + |bm25|)`, no `distance`)
- `get_session_memories` — chronological turn recall by `sessionId`
- `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` — `§10.7` standalone `npx` Memory Core placement row
<!-- /binding-inventory -->

---

## Acceptance Criteria

- [ ] **AC-1:** `standaloneConfigBase.mjs`, `standalone.mjs`, and `standaloneServer.mjs` boot cleanly over a temporary `NEO_PLANE_DATA_ROOT` passing `assertPlaneCoherence`, `assertPlaneMemberCoherence`, and `derivePlaneMemberPaths` completeness over the 3 standalone plane members (`memoryWal.dirProd`, `standaloneIndexPath`, `logPath`) without inheriting Tier-1 full-plane members, enforce standalone placement + `'stdio'` transport on both launcher and `--serve` entry paths, and `0019-aiconfig-reactive-provider-ssot.md` `§10.7` records the standalone `npx` Memory Core row (`npm run ai:lint-config-template-ssot` / config parity specs green).
- [ ] **AC-2:** `standaloneWorkspaceKey.mjs` deterministically resolves `explicit-slug -> git-remote -> git-toplevel -> cwd` (normalizing SSH and HTTPS `origin` URLs identically to `<lowercase-host>/<owner>/<repo>`), and `healthcheck` reports `{plane: {id, dataRoot}, workspace, workspaceSource, storage: {walBytes, indexBytes, memoryCount, retentionPolicy: 'none-v1'}}`.
- [ ] **AC-3:** `openapi.yaml` (`x-neo-exact-tool-profiles.profiles.standalone`) and `standaloneToolService.mjs` expose **only** the 6 approved tools (`tools/list` count === 6, non-profile tools rejected fail-closed), MCP `initialize` `instructions` directs session-start `query_recent_turns` + turn-end `add_memory`, and `standaloneToolService.mjs` / `standaloneServer.mjs` import neither `better-sqlite3`, `chromadb`, nor `@chroma-core/default-embed`.
- [ ] **AC-4:** `add_memory` writes canonical `wal-YYYY-MM-DD.jsonl` (without unverified `agentIdentity`) while leaving **both** `.embedded.jsonl` and `.graph.jsonl` absent; `standaloneSqliteIndex.mjs` serves workspace-scoped `query_recent_turns` (`summaryFallback: true`), `query_raw_memories` (`retrievalMode: 'lexical-fts5'`, `bm25Score`, monotonically non-increasing `lexicalRankScore: |bm25| / (1 + |bm25|)`, no `distance`/`compositeScore`), and `get_session_memories`, catches up concurrent WAL appends via the per-read `stat.size` byte watermark without restart, and rebuilds `index.sqlite` losslessly when deleted.

---

## Out of Scope

- Rostered-owner WAL promotion CLI/helper (`promote` subcommand) into a full plane (owned by Leaf 2 `#1717`).
- `packMcpServer.mjs` npm tarball assembly, `server.json` `packages`, and npm/MCP Registry publishing (owned by Leaves 3 and 6).
- Standalone WAL/index retention or compaction (`retentionPolicy: 'none-v1'`).

---

## Avoided Traps

- **Calling `BaseServer#collectMemberEntries` (`BaseServer.mjs:725-738`):** Concatenates all 12 `TIER1_PLANE_MEMBER_PATHS` onto local paths; `standaloneServer#getPlaneMembers()` must invoke `collectPlaneMembers` directly over `STANDALONE_PLANE_MEMBER_PATHS`.
- **Importing `ai/mcp/server/memory-core/toolService.mjs`:** Statically pulls `GraphService` (`better-sqlite3`) and `MemoryService` (`chromadb`) into the import graph; `standaloneToolService.mjs` binds `ToolService` directly.
- **Double-quoted string literals in SQLite DML:** Bundled SQLite `3.51+` compiles with `DQS_DML=0`; all SQL statements in `standaloneSqliteIndex.mjs` must use single-quoted SQL literals and `?` parameter bindings.

---

## Related

- Parent Epic: #1715
- Source Discussion: [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941)
- Sibling precedent: `ai/mcp/server/neural-link/standalone.mjs` (`neomjs/neo-agent-brain#1282`)

Origin Session ID: `2f084437-cc03-4b7b-aace-a27bff4aa3f9`
Retrieval Hint: `"standaloneSqliteIndex" "standaloneWorkspaceKey" "standaloneConfigBase" "x-neo-exact-tool-profiles" "ADR-0019 10.7"`

## Timeline

- 2026-11-14T17:54:22Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-14T17:54:23Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T17:54:23Z @neo-gemini-julia added the `ai` label
- 2026-11-14T17:54:23Z @neo-gemini-julia added the `architecture` label
- 2026-11-14T17:54:23Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T17:55:07Z @neo-gemini-julia added parent issue #1715
- 2026-11-14T17:55:19Z @neo-gemini-julia marked this issue as blocking #1717
- 2026-11-14T17:55:22Z @neo-gemini-julia marked this issue as blocking #1718
- 2026-11-14T18:30:56Z @neo-gemini-julia cross-referenced by PR #1721
- 2026-11-14T19:54:08Z @neo-gemini-julia referenced in commit `440f8b3` - "fix(memory-core): address round-1 review findings on standalone server (#1716)"

