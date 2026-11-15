---
id: 1717
title: Add rostered-owner standalone WAL promotion helper and RLS spec
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - security
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-14T17:54:25Z'
updatedAt: '2026-11-15T06:33:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1717'
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
  - '[x] 1716 Add standalone memory-core MCP server, SQLite FTS5 index, and ADR'
blocking:
  - '[ ] 1718 Generalize packMcpServer for neo-memory-core and cross-harness walk'
---
# Add rostered-owner standalone WAL promotion helper and RLS spec

## Context

Graduated as **Leaf 2** of Parent Epic [`neomjs/neo-agent-brain#1715`](https://github.com/neomjs/neo-agent-brain/issues/1715) from [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941) (`STEP_BACK` Point 2 `✗` at `DC_kwDODSospM4BK1En` + quorum carry-over at `DC_kwDODSospM4BK1Hk`).

Standalone `npx neo-memory-core` writes unmarked `wal-YYYY-MM-DD.jsonl` segments with no authentication (`userId` and `agentIdentity` absent). When an adopter upgrades to a full `neomjs/neo-agent-brain` plane, promoting those segments directly without stamping a validated rostered owner makes every promoted memory world-readable on team planes (`GraphService#isRlsVisible` returns `true` when `properties.userId == null`) and invisible to `@me` owner recall.

- `Live latest-open + closed sweep:` checked latest 20 open issues (`155` open — red backlog band) + closed search (`promoteStandaloneWal memoryWalStore isRlsVisible`) on `neomjs/neo-agent-brain` at `2026-11-14T17:50:16Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 `all`-status messages at `2026-11-14T17:50:16Z`; no competing claim found.
- `MC sweep:` `query_raw_memories` (`memoryWalStore isRlsVisible userId null promotion`), surfaced `DC_kwDODSospM4BK1En` only.
- `Own-assignment sweep:` checked open `@me` issues (`#1548` and `#1230` — both orthogonal).
- `Structure-map gate:` executed `npm run ai:structure-map -- --files --loc` (`exit 0`); owning folder verified at `ai/services/memory-core/helpers/`.
- `unowned-rationale: Ready for a contributor; maintainers provide planning and review.` (`blocked_by` Leaf 1).

---

## The Problem

1. **World-readable RLS leak (`GraphService.mjs:30-48`):** `isRlsVisible` treats `ownerUserId == null` as visible to every requester, while `@me` scoping in `query_recent_turns` filters by `agentIdentity`. Unstamped standalone records merged into a team plane leak across identities and vanish from the owner's `@me` feed.
2. **Free-text `--owner` bypass (`DC_kwDODSospM4BK1Hk`):** Accepting an arbitrary unrostered `--owner` string stamps an unheld handle that no rostered seat owns on the target plane, and reading `ai/graph/identityRoots.mjs` at runtime validates the Neo maintainer roster rather than an adopter's own plane roster (`#874`).
3. **Same-day UTC segment collisions (`F2`):** WAL segments are keyed by UTC date (`wal-YYYY-MM-DD.jsonl`). Copying files over an active plane's `memoryWal` directory would overwrite existing same-day plane records unless merged idempotently by record `id`.

---

## The Architectural Reality

- Design authority: n/a — adds the standalone-to-plane WAL promotion helper graduated in Discussion #20941 so promoted records conform to existing `GraphService#isRlsVisible` and `memoryWalStore` contracts.
- `ai/services/memory-core/GraphService.mjs:30-48` (`isRlsVisible`) enforces tenant visibility via `properties.userId` / `properties.agentIdentity`.
- `ai/graph/normalizeAgentIdentityNodeId.mjs:1-27` normalizes `@<handle>` / `AGENT:<handle>` identities, `ai/graph/mailboxRoots.mjs` defines `SYSTEM_SENDER` and `BROADCAST_SENTINEL`, and `AgentIdentity` nodes in the target plane's SQLite graph (`Nodes` where `json_extract(data, '$.label') = 'AgentIdentity'`) define that plane's rostered identities (`#874`: runtime Agent OS / MCP helpers must never read `ai/graph/identityRoots.mjs`).
- `ai/services/memory-core/helpers/memoryWalStore.mjs:122-185` tracks pending embed (`wal-YYYY-MM-DD.embedded.jsonl`, `:131`) and pending graph projection (`wal-YYYY-MM-DD.graph.jsonl`, `:140`) separately from `wal-YYYY-MM-DD.jsonl`, while preserving each record's `planeId` provenance.
- Structural Pre-Flight (`Stage 1` fast-path): `ai/services/memory-core/helpers/promoteStandaloneWal.mjs` matches sibling `ai/services/memory-core/helpers/memoryWalStore.mjs`.

---

## The Fix

1. **Author `ai/services/memory-core/helpers/promoteStandaloneWal.mjs`:**
   - Require `--target-graph-db <path>` and `--owner <agentIdentity>` (fail closed before any target write if `--target-graph-db` is missing or unreadable, or if `--owner` is missing, `SYSTEM_SENDER`, `BROADCAST_SENTINEL`, or not present in the target plane's `AgentIdentity` nodes read from `--target-graph-db` via `node:sqlite`; never read `ai/graph/identityRoots.mjs`).
   - Default `userId` to the normalized `@<owner>` handle when `--user-id` is omitted; support optional `--workspace <ws>` filter (matching via `standaloneWorkspaceKey.mjs` when a path is passed).
   - Stamp `agentIdentity` and `userId` (plus top-level `metadata.agentIdentity` / `metadata.userId` parity expected by WAL/graph projection) while preserving original `id`, `timestamp`, `sessionId`, `workspace`, and standalone `planeId` provenance.
   - Merge records idempotently by `id` into target `wal-YYYY-MM-DD.jsonl` segments under `walAppendLock` (never overwriting existing target records, skipping already-present `id`s on re-run, and writing **neither** `.embedded.jsonl` nor `.graph.jsonl` markers).
2. **Wire `neo-memory-core promote` CLI subcommand in `ai/mcp/server/memory-core/standalone.mjs`:**
   - Parse `neo-memory-core promote --target-wal-dir <dir> --target-graph-db <path> --owner <agentIdentity> [--source-wal-dir <dir>] [--user-id <userId>] [--workspace <ws>]` and print a structured JSON summary (`{promotedCount, skippedDuplicateCount, segmentsTouched, owner, userId}`).
3. **Add Playwright Unit/Integration Spec (`test/playwright/unit/ai/services/memory-core/helpers/promoteStandaloneWal.spec.mjs`):**
   - Verify: (a) missing/unreadable `--target-graph-db` or missing, sentinel, or unrostered `--owner` throws a descriptive error before mutating target WAL files; (b) same-day UTC segment merge against a fixture target graph preserves existing plane records and deduplicates repeated runs by `id` while leaving `.embedded.jsonl` and `.graph.jsonl` untouched; (c) promoted records carrying the standalone `planeId` project into `GraphService` and pass `isRlsVisible` for `--owner` (`@me` recall + search) while remaining invisible to a second rostered identity.

---

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `promoteStandaloneWal` (`ai/services/memory-core/helpers/promoteStandaloneWal.mjs`) | `GraphService.mjs:30-48`, `SQLite.mjs` (`AgentIdentity` nodes), `#874`, `Discussion #20941` `STEP_BACK` #2 | Validates `--owner` exclusively against `AgentIdentity` nodes in `--target-graph-db`, stamps `agentIdentity` + `userId`, merges `wal-YYYY-MM-DD.jsonl` idempotently by `id`, leaves `.embedded.jsonl` & `.graph.jsonl` unwritten | Refuses missing/unreadable `--target-graph-db` or unrostered/missing `--owner` fail-closed before any write | CLI `--help` + JSDoc | `promoteStandaloneWal.spec.mjs` |
| `neo-memory-core promote` (`ai/mcp/server/memory-core/standalone.mjs`) | `Discussion #20941` `§4.1` | CLI entrypoint invoking `promoteStandaloneWal` and printing JSON promotion receipt | Exits non-zero on invalid args, missing/unreadable `--target-graph-db`, or unrostered owner | Package `README.md` (Leaf 3) | `promoteStandaloneWal.spec.mjs` |

- **Decision Record:** `Optional: neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`
- **Decision Record impact:** `aligned-with neomjs/neo-agent-brain#adr:0019-aiconfig-reactive-provider-ssot`

---

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: standalone-memory-core-contract -->
- `ai/services/memory-core/helpers/promoteStandaloneWal.mjs` — target-plane SQLite `AgentIdentity` rostered-owner validation, `agentIdentity`/`userId` RLS stamping, and idempotent by-`id` UTC WAL segment merger
- `neo-memory-core promote` — CLI subcommand exposing required `--target-wal-dir`, `--target-graph-db`, and `--owner`, plus optional `--source-wal-dir`, `--data-root`, `--user-id`, and `--workspace`
<!-- /binding-inventory -->

---

## Acceptance Criteria

- [ ] **AC-1:** `promoteStandaloneWal.mjs` requires `--target-graph-db` and refuses missing/unreadable `--target-graph-db` or missing, sentinel (`@system`, `AGENT:*`), or unrostered `--owner` values solely against `AgentIdentity` nodes in `--target-graph-db` (without reading `ai/graph/identityRoots.mjs`), performing zero target-directory writes on refusal.
- [ ] **AC-2:** Promoting standalone `wal-YYYY-MM-DD.jsonl` segments into a target WAL directory already containing same-day records (validated against a fixture target SQLite graph) merges by `id` without overwriting existing records, is idempotent on re-run (`promotedCount === 0`, `skippedDuplicateCount === N`), honors optional `--workspace` filtering, preserves standalone `planeId` provenance, and writes neither `.embedded.jsonl` nor `.graph.jsonl`.
- [ ] **AC-3:** `neo-memory-core promote` CLI subcommand works end-to-end from `standalone.mjs`, and `promoteStandaloneWal.spec.mjs` proves both post-projection recall of records carrying the standalone `planeId` and `GraphService#isRlsVisible` tenant isolation (visible to `--owner`, hidden from a second rostered identity).

---

## Out of Scope

- Mutating ChromaDB collections directly from `promoteStandaloneWal.mjs` (the target plane's existing WAL embed and graph-projection drains own ingestion of unmarked segments).
- Deleting source standalone WAL segments during promotion (promotion is non-destructive).

---

## Avoided Traps

- **Copying segment files with `fs.copyFile`:** Clobbers existing same-day UTC segments in an active plane.
- **Accepting arbitrary `--owner` strings or falling back to `ai/graph/identityRoots.mjs`:** Reopens the RLS isolation gap under an unrostered handle (`DC_kwDODSospM4BK1Hk`) or checks the Neo team's seed list instead of the adopter's target plane (`#874`).

---

## Related

- Parent Epic: #1715
- Source Discussion: [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941)

Origin Session ID: `2f084437-cc03-4b7b-aace-a27bff4aa3f9`
Retrieval Hint: `"promoteStandaloneWal" "isRlsVisible" "memoryWalStore" "neo-memory-core promote"`

## Timeline

- 2026-11-14T17:54:26Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T17:54:26Z @neo-gemini-julia added the `good first issue` label
- 2026-11-14T17:54:26Z @neo-gemini-julia added the `help wanted` label
- 2026-11-14T17:54:27Z @neo-gemini-julia added the `ai` label
- 2026-11-14T17:54:27Z @neo-gemini-julia added the `security` label
- 2026-11-14T17:54:27Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T17:55:10Z @neo-gemini-julia added parent issue #1715
- 2026-11-14T17:55:19Z @neo-gemini-julia marked this issue as being blocked by #1716
- 2026-11-14T17:55:25Z @neo-gemini-julia marked this issue as blocking #1718
- 2026-11-14T19:25:09Z @neo-opus-ada cross-referenced by PR #1721
- 2026-11-15T05:45:08Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-15T06:04:31Z @neo-gemini-julia cross-referenced by PR #1752
- 2026-11-15T06:39:20Z @neo-gemini-julia referenced in commit `208fa2b` - "fix(memory-core): require target graph DB for rostered WAL promotion (#1717)"

