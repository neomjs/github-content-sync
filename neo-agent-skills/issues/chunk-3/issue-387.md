---
id: 387
title: Document standalone neo-memory-core workflow for app-audience skills
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-14T17:54:32Z'
updatedAt: '2026-11-14T17:54:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/387'
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
blocking: []
---
# Document standalone neo-memory-core workflow for app-audience skills

## Context

Graduated as **Leaf 5** (`neomjs/neo-agent-skills`) of Parent Epic [`neomjs/neo-agent-brain#1715`](https://github.com/neomjs/neo-agent-brain/issues/1715) from [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941) (`E1` + `OQ2` + `STEP_BACK` Point 2⚠).

In `neomjs/neo-agent-skills#312`, `.agents/skills/context-recovery/references/standalone-context-recovery.md` was authored for `--audience app` workspaces assuming **no** Memory Core MCP server exists outside maintainer checkouts (instructing agents to reconstruct solely from `git status` and GitHub CLI). With standalone `neo-memory-core` (`npx -y neo-memory-core`) available to any `--audience app` workspace, `standalone-context-recovery.md` should detect when standalone `memory-core` is mounted (the 6-tool exact profile) and run `query_recent_turns` / `get_session_memories` / `query_raw_memories` (`lexical-fts5`) before falling back to git-only recovery — without bloating the top-level `SKILL.md` router (`create-skill` Progressive Disclosure rule).

- `Live latest-open + closed sweep:` checked latest 25 open issues (`36` open — orange backlog band) + closed search (`standalone-context-recovery memory-core`, `Out of Scope` / `Avoided Traps` read on `#312` and `#211`) on `neomjs/neo-agent-skills` at `2026-11-14T17:50:16Z`; no equivalent or prior exclusion found.
- `A2A in-flight claim sweep:` checked latest 30 `all`-status messages at `2026-11-14T17:50:16Z`; no competing claim found.
- `MC sweep:` `query_raw_memories` (`standalone-context-recovery neo-agent-skills audience app`), surfaced `#312` precedent.
- `Own-assignment sweep:` `0` open assigned issues on `neomjs/neo-agent-skills`.
- `Meta-skill & structure-map check:` consulted `.agents/skills/create-skill/SKILL.md` (Progressive Disclosure verified: top-level `context-recovery/SKILL.md` stays a 10-line router; depth lives in `references/standalone-context-recovery.md`); `ai:structure-map` N/A in `neomjs/neo-agent-skills`.
- `unowned-rationale: Ready for a contributor; maintainers provide planning and review.` (Labeled `enhancement`, `ai`, `agent-os`, `documentation`, `good first issue`, `help wanted`).

---

## The Problem

`.agents/skills/context-recovery/references/standalone-context-recovery.md:1-54` currently states that standalone `--audience app` workspaces have no Memory Core and skips MCP recency/search tools completely. When an adopter enables `npx -y neo-memory-core` (or `npx neo-app --memory`), an agent reading `standalone-context-recovery.md` after compaction ignores its own `memory-core` MCP server — or, if it calls `query_raw_memories`, may mistake a BM25 lexical synonym miss (`retrievalMode: 'lexical-fts5'`) for absence or attempt to call full-plane-only tools (`list_messages`, `get_context_frontier`, `query_summaries`).

---

## The Architectural Reality

- Design authority: n/a — extends `standalone-context-recovery.md` with an optional branch when standalone `neo-memory-core` is mounted, preserving git-only recovery when no memory MCP server is present.
- `.agents/skills/skills.manifest.json:37-41` already ships `context-recovery` to `audiences: ["maintainer", "app"]`.
- `.agents/skills/context-recovery/SKILL.md:1-10` routes `--audience app` workspaces to `.agents/skills/context-recovery/references/standalone-context-recovery.md` (10 lines; must remain compact per `create-skill` and `turn-memory-pre-flight`).
- Standalone `neo-memory-core` (`Discussion #20941` `OQ2`) exposes **only** 6 tools (`healthcheck`, `get_mcp_tool_handbook`, `add_memory`, `query_recent_turns`, `query_raw_memories`, `get_session_memories`), where `query_recent_turns` (`detail: 'summary'`) returns `summaryFallback: true` excerpts and `query_raw_memories` runs `node:sqlite` FTS5 BM25 (`retrievalMode: 'lexical-fts5'`, `bm25Score`, `lexicalRankScore`) rather than vector embeddings.

---

## The Fix

1. **Update `.agents/skills/context-recovery/references/standalone-context-recovery.md`:**
   - Keep `context-recovery/SKILL.md` <= 12 lines (updating only the 1-line parenthetical if needed so `--audience app` routes to `standalone-context-recovery.md` whether or not standalone `neo-memory-core` is mounted, while full Agent OS maintainer workspaces with `list_messages`/`get_context_frontier` use `context-recovery-workflow.md`).
   - Add a concise **Step 0 (When standalone `memory-core` MCP is mounted)** in `standalone-context-recovery.md`:
     1. Call `query_recent_turns({limit: 10, detail: 'summary'})` (noting `summaryFallback: true` deterministic excerpts) and `get_session_memories` when recovering a specific `sessionId`.
     2. Run 1–2 keyword-targeted `query_raw_memories` queries (explicitly noting `retrievalMode: 'lexical-fts5'` BM25 lexical matching so agents vary exact identifiers/nouns on a miss instead of assuming a concept was never saved, and never call absent full-plane tools like `query_summaries`, `list_messages`, or `get_context_frontier`).
     3. Save turn outcomes via `add_memory` after completing work so cross-harness sessions stay synchronized.
   - Preserve the existing git + public GitHub recovery sequence unchanged when no `memory-core` MCP server is mounted, and keep total file length tight (`<= 90` lines, respecting `AGENTS.md §self_evolving_systems` substrate-size discipline).
2. **Verify Skill Tests & Guards:**
   - Run `npm test` in `neomjs/neo-agent-skills` to confirm manifest, generated `AGENTS.md` (if touched), and skill lints pass.

---

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/context-recovery/references/standalone-context-recovery.md` | `Discussion #20941` `E1` / `OQ2`, `create-skill` | Uses standalone 6-tool `memory-core` (`query_recent_turns`, `query_raw_memories` `lexical-fts5`, `get_session_memories`, `add_memory`) when mounted | Existing git/GitHub recovery when `memory-core` is not mounted | `standalone-context-recovery.md` | `npm test` in `neomjs/neo-agent-skills` |

- **Decision Record:** `Optional: neomjs/neo#adr:0007-agents-md-compaction-taxonomy`
- **Decision Record impact:** `aligned-with neomjs/neo#adr:0007-agents-md-compaction-taxonomy` (keeps top-level `SKILL.md` router unchanged in size; adds bounded branch inside on-demand `--audience app` reference file; retirement trigger: if standalone `neo-memory-core` ever gains local embeddings, retire the `lexical-fts5` synonym caveat).

---

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: standalone-memory-core-contract -->
- `.agents/skills/context-recovery/references/standalone-context-recovery.md` — documents the 6-tool standalone `neo-memory-core` recovery & turn-save branch (`lexical-fts5` + `summaryFallback: true`) alongside the zero-MCP git fallback
<!-- /binding-inventory -->

---

## Acceptance Criteria

- [ ] **AC-1:** `.agents/skills/context-recovery/references/standalone-context-recovery.md` documents the standalone `neo-memory-core` branch (`query_recent_turns`, `get_session_memories`, `query_raw_memories` `lexical-fts5` keyword guidance, and `add_memory` persistence) while keeping the zero-MCP git/GitHub path intact and keeping `context-recovery/SKILL.md` compact (`<= 12` lines).
- [ ] **AC-2:** `standalone-context-recovery.md` explicitly warns against calling full-plane-only tools (`query_summaries`, `list_messages`, `get_context_frontier`) in standalone mode and stays `<= 90` lines.
- [ ] **AC-3:** `npm test` in `neomjs/neo-agent-skills` exits `0`.

---

## Out of Scope

- Modifying maintainer `context-recovery-workflow.md` (`neo-mjs-memory-core` full plane).
- Adding new top-level skills to `skills.manifest.json`.

---

## Avoided Traps

- **Putting standalone `neo-memory-core` instructions into turn-loaded `AGENTS.md` or top-level `SKILL.md`:** Violates `ADR 0007` Progressive Disclosure by taxing workspaces that do not mount `neo-memory-core`.

---

## Related

- Parent Epic: neomjs/neo-agent-brain#1715
- Source Discussion: [`neomjs/neo#20941`](https://github.com/orgs/neomjs/discussions/20941)
- Prior standalone recovery ticket: [`neomjs/neo-agent-skills#312`](https://github.com/neomjs/neo-agent-skills/issues/312)

Origin Session ID: `2f084437-cc03-4b7b-aace-a27bff4aa3f9`
Retrieval Hint: `"standalone-context-recovery" "lexical-fts5" "neo-memory-core" "audience app"`

## Timeline

- 2026-11-14T17:54:33Z @neo-gemini-julia added the `documentation` label
- 2026-11-14T17:54:33Z @neo-gemini-julia added the `enhancement` label
- 2026-11-14T17:54:33Z @neo-gemini-julia added the `good first issue` label
- 2026-11-14T17:54:34Z @neo-gemini-julia added the `help wanted` label
- 2026-11-14T17:54:34Z @neo-gemini-julia added the `ai` label
- 2026-11-14T17:54:34Z @neo-gemini-julia added the `agent-os` label
- 2026-11-14T17:55:33Z @neo-gemini-julia added parent issue #1715

