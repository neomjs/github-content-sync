---
id: 449
title: Add plugins/context-compaction-memory magnet and standalone fallbacks
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-24T22:00:35Z'
updatedAt: '2026-11-24T23:51:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/449'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 446
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 456 Add upstream-skill-feedback skill and issue form for plugin sessions'
  - '[x] 447 Generalize generate-claude-plugin for multi-plugin marketplace'
blocking: []
---
# Add plugins/context-compaction-memory magnet and standalone fallbacks

## Context

Model-Magnet Leaf 2 (`plugins/context-compaction-memory`) under parent epic #446. After `/compact` or across fresh sessions, coding models lose exact branch/ticket/decision state to lossy summaries. Per `@neo-opus-grace`'s live `SearchPlugins` probe (`#446` `issuecomment-7146202696`), generic single-agent hosted memory plugins already crowd `"persistent memory"`, whereas our distinct capability is **reconstructing state from structured turn records rather than lossy summaries, local append-only storage with zero external account, and cross-agent / cross-model-family memory**.

`handoff: @neo-opus-grace` (claimed for next pickup by `@neo-opus-grace` per `MESSAGE:11495afa`; `good first issue` / `help wanted` omitted for coordinated rollout).

## The Problem

1. `session-sunset`, `memory-mining`, and `turn-memory-pre-flight` currently assume full-plane Agent OS MCP tools (`add_message` self-DM, `query_summaries`, `ask_knowledge_base`) without standalone (`gh` / `git log` / local `.neo-ai/handover.md`) execution branches and honest capability-step notes explaining what local `neo-memory-core` adds once published.
2. `npm view neo-memory-core` currently returns `E404` until `neomjs/neo-agent-brain#1719` publishes to npm (`issuecomment-7146202696` Stage 3.3): `#449` must NOT ship a dead `claude mcp add neo-memory-core -- npx -y neo-memory-core` first step before `brain#1719` lands.

## The Architectural Reality

- Owning files: `.agents/skills/session-sunset/references/`, `.agents/skills/memory-mining/references/`, `.agents/skills/turn-memory-pre-flight/references/`, `scripts/generate-claude-plugin.mjs` (`PLUGIN_BUNDLES`), `.claude-plugin/marketplace.json`, `plugins/context-compaction-memory/`, and plugin eval cases.
- Scope boundary with `#387` & `brain#1719`: `#387` owns the 6-tool `neo-memory-core` branch inside `standalone-context-recovery.md`, and `neomjs/neo-agent-brain#1719` owns publishing `neo-memory-core` to npm. This leaf ships zero-MCP standalone branches that work immediately in any git/`gh` workspace and gates any `npx -y neo-memory-core` install command on `brain#1719`.
- Design authority: n/a — adds opt-in standalone execution branches to `session-sunset`, `memory-mining`, and `turn-memory-pre-flight` references and projects `plugins/context-compaction-memory`.
- Structure-map & meta-skill gate (`§1b`, `§1c`): consulted `.agents/skills/create-skill/SKILL.md` (`ADR 0008`) and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`).

## The Fix

1. Add concise standalone execution branches + honest capability-step degradation notes (Directory Policy §2D/§4C compliant) inside `session-sunset` (structured handover file / git state when A2A & Memory Core are absent), `memory-mining` (`git log -S` / `gh issue list --search` when semantic turn memory is absent, stating clearly that turn-diary recall across compacted sessions requires Memory Core), and `turn-memory-pre-flight` (`CLAUDE.md`, `AGENTS.md`, `.claude/skills/**` Map-vs-Atlas byte budgets in any repo).
2. Register `context-compaction-memory` in `PLUGIN_BUNDLES`:
   - `description` (`<= 500` chars) leading with the moment after `/compact` or session handover: reconstructing exact active branch, PR, and decision state from structured records instead of lossy summaries, keeping `CLAUDE.md` / `AGENTS.md` lean, with local account-free storage.
   - `skills: ['context-recovery', 'session-sunset', 'memory-mining', 'turn-memory-pre-flight', 'upstream-skill-feedback']` (`contained` reach; omit `npx -y neo-memory-core` setup line until `neomjs/neo-agent-brain#1719` publishes on npm, or gate behind `brain#1719`).
3. Materialize `plugins/context-compaction-memory/` + `.claude-plugin/marketplace.json`, add checked-in eval cases + fixed `SearchPlugins` query set (`after compact`, `context compaction`, `session handoff`, `context recovery`), and verify `npm run generate-claude-plugin -- --check` + `claude plugin validate ./plugins/context-compaction-memory`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `session-sunset`, `memory-mining`, `turn-memory-pre-flight` standalone branches | `ADR 0007`, `ADR 0008`; Directory Policy §2D/§4C | Works end-to-end in any repo via `git`/`gh` + structured handover files, stating honestly at recall/handover steps what Memory Core adds | Full Agent OS maintainer path unchanged when `neo-mjs-memory-core` is mounted | `plugins/context-compaction-memory/README.md` | `npm run lint && npm test` |
| `plugins/context-compaction-memory/` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review | Bundles `context-recovery`, `session-sunset`, `memory-mining`, `turn-memory-pre-flight`, and `upstream-skill-feedback` (`contained` reach, `description <= 500` chars, zero dead `npx` commands) | Fails closed in `generate-claude-plugin.mjs --check` | `plugins/context-compaction-memory/README.md` | `--check` + `claude plugin validate` + checked-in eval cases beating no-plugin baseline |

Decision Record impact: aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `session-sunset`, `memory-mining`, and `turn-memory-pre-flight` reference payloads include standalone (`git`/`gh` + structured handover) branches with honest step-level capability notes while keeping `SKILL.md` routers within budget and `contained` reach.
- [ ] `plugins/context-compaction-memory/` is registered in `PLUGIN_BUNDLES` (`description <= 500` chars pitched at post-`/compact` turn-record recovery & lean `CLAUDE.md`, bundling `context-recovery`, `session-sunset`, `memory-mining`, `turn-memory-pre-flight`, and `upstream-skill-feedback`, with zero `E404` `npx` commands prior to `neomjs/neo-agent-brain#1719`) and passes `npm run generate-claude-plugin -- --check` and `claude plugin validate ./plugins/context-compaction-memory`.
- [ ] Checked-in eval cases (`claude plugin eval` / fresh-session compaction-recovery benchmark) beat the no-plugin baseline arm, and the fixed `SearchPlugins` query set is recorded (`[L3-deferred — post-listing directory search probe tracked on #446]`).
- [ ] `npm run lint && npm test` exit `0` with bumped `package.json` + `package-lock.json`.

## Out of Scope

- Publishing `neo-memory-core` to npm (`neomjs/neo-agent-brain#1719`) or duplicating `#387` without coordination.

## Avoided Traps

- **Documenting `npx -y neo-memory-core` before `neomjs/neo-agent-brain#1719` is live on npm (`E404` today).**
- **Pitching generic single-agent `"persistent memory"` or tuning `keywords` instead of `description` + skill names.**

## Related

- Parent epic: #446
- Prerequisites: #447 (`PR #452`), #453 (`PR #454`), #455, and the `upstream-skill-feedback` sub-ticket
- Sibling / npm gate: `#387`, `neomjs/neo-agent-brain#1715`, `neomjs/neo-agent-brain#1719`

Live latest-open + closed sweep: checked latest 20 open issues (`#289`–`#445`, `44` open total, orange band `25–50`) and open/closed search (`context-recovery OR session-sunset OR memory-mining`, `#387`, `#289`, `#312`) at `2026-11-24T21:56Z`; `#387` scoped and preserved without overlap.
A2A in-flight claim sweep: latest 30 `all`-status messages checked at `2026-11-24T21:54Z`; zero competing claims.
MC sweep: `query_raw_memories` on `"plugin catalog nobody searches neomjs skills invisible standalone MCP"` (`5` hits) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "plugins/context-compaction-memory session-sunset memory-mining turn-memory-pre-flight after /compact"

## Timeline

- 2026-11-24T22:00:36Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T22:00:37Z @neo-gemini-julia added the `ai` label
- 2026-11-24T22:00:37Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T22:00:37Z @neo-gemini-julia added the `agent-os` label
- 2026-11-24T22:01:11Z @neo-gemini-julia added parent issue #446
- 2026-11-24T22:02:57Z @neo-gemini-julia marked this issue as being blocked by #447
- 2026-11-24T22:25:03Z @neo-opus-grace cross-referenced by #446
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:53:19Z @neo-gemini-julia marked this issue as being blocked by #456

