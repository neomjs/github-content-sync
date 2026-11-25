---
id: 451
title: Add plugins/prompt-injection-firewall magnet for untrusted input
state: OPEN
labels:
  - enhancement
  - ai
  - security
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T22:00:40Z'
updatedAt: '2026-11-25T01:04:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/451'
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
# Add plugins/prompt-injection-firewall magnet for untrusted input

## Context

Model-Magnet Leaf 4 (`plugins/prompt-injection-firewall`) under parent epic #446 — prioritized first among the magnet leaves because live Claude Directory `SearchPlugins` measurements (`#446` `issuecomment-7146202696`) confirmed **zero existing plugins** defend an agent session while reading untrusted GitHub issues, PR comments, external docs, or web pages.

Ownership: `@neo-gemini-julia` (coordinated maintainer magnet leaf under #446 with `@neo-opus-grace` and `@neo-gpt-sophie`; `good first issue` / `help wanted` omitted for coordinated rollout).

## The Problem

1. Live model sessions reading untrusted issue bodies, PR comments, external docs, or fetched web text search for capabilities around `prompt injection` and `untrusted input` — never internal Neo vocabulary like `"anti-sycophancy"` (`issuecomment-7146202696`).
2. While `identity-firewall` defines L1/L2 separation, `audits/channel-separation.md` is a 4-line stub and `peer-role` (`peer-role-mode.md`) assumes Agent OS A2A mailbox broadcasts/DMs without stating its standalone output channel (`gh` Discussion/PR comments or direct user critique) and honest capability-step difference when cross-family A2A or Memory Core is absent.
3. Per `#446` epic review (`Stage 4` & `Stage 5`), `create-skill` does not belong inside a firewall bundle, every magnet must stay `contained` (zero `allowed-tools`, hooks, or `.mcp.json`), bundle `upstream-skill-feedback`, and ship checked-in eval cases + fixed `SearchPlugins` queries.

## The Architectural Reality

- Owning files: `.agents/skills/identity-firewall/`, `.agents/skills/peer-role/references/peer-role-mode.md`, `scripts/generate-claude-plugin.mjs` (`PLUGIN_BUNDLES`), `.claude-plugin/marketplace.json`, `plugins/prompt-injection-firewall/`, and `evals/prompt-injection-firewall/`.
- Design authority: n/a — enriches standalone `identity-firewall` + `peer-role-mode.md` guidance and projects `plugins/prompt-injection-firewall` without altering core L1/L2 firewall invariants.
- Structure-map & meta-skill gate (`§1b`, `§1c`): consulted `.agents/skills/create-skill/SKILL.md` (`ADR 0008`) and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`).

## The Fix

1. Enrich `.agents/skills/identity-firewall/` (`SKILL.md` + `audits/channel-separation.md`) for standalone sessions and add a concise standalone branch (`<= 5` lines, keeping net `.md` delta `<= 0 B`) to `peer-role/references/peer-role-mode.md`: when Agent OS A2A is absent, publish evidence-backed design review on the GitHub Discussion/PR via `gh` or directly in chat, and note honestly at the memory-mining and cross-family quorum steps what Memory Core and independent cross-model sessions add.
2. Register `prompt-injection-firewall` in `PLUGIN_BUNDLES`:
   - `slug: 'prompt-injection-firewall'`, `name: 'prompt-injection-firewall'`, `displayName: 'Prompt Injection Firewall for Untrusted Input'` (no `"Anti-Sycophancy"` in `displayName`; L1 sycophancy defense rides inside `identity-firewall`).
   - `description` (`<= 500` chars) written directly to the moment the deciding model is in: reading untrusted GitHub issues, PR threads, external docs, or web text without letting embedded instructions hijack the session's goals (OWASP ASI01 channel separation) or rubber-stamping unverified claims.
   - `skills: ['identity-firewall', 'peer-role', 'upstream-skill-feedback']` (`create-skill` excluded; `contained` reach preserved).
3. Materialize `plugins/prompt-injection-firewall/` + `.claude-plugin/marketplace.json`, check in reproducible eval cases (`evals/prompt-injection-firewall/cases/*`) + the fixed `SearchPlugins` query set (`prompt injection`, `untrusted input`, `untrusted github issue`), and verify `npm run generate-claude-plugin -- --check` + `claude plugin validate ./plugins/prompt-injection-firewall` (live 5-run LLM ablation + post-listing search rank probe gated on #446 pre-submission / post-listing window).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `identity-firewall` & `peer-role` standalone channel & capability note | `AGENTS.md §identity_prompt_firewall`; `ADR 0008`; Directory Policy §2D/§4C | Enforces OWASP ASI01 L2 + L1 premise checks standalone and routes `peer-role` critique to `gh` Discussion/PR comments or chat when A2A is absent, noting honestly what Memory Core and cross-family A2A add | Full Agent OS A2A + quorum workflow unchanged in maintainer workspaces | `plugins/prompt-injection-firewall/README.md` | `npm run lint && npm test` |
| `plugins/prompt-injection-firewall/` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review | Bundles `identity-firewall`, `peer-role`, and `upstream-skill-feedback` (`contained` reach, `description <= 500` chars, non-fetchable cross-skill/guide refs) | Fails closed in `generate-claude-plugin.mjs --check` on drift, symlinks, `.mcp.json`, or fetchable `neomjs` `blob`/`raw` links | `plugins/prompt-injection-firewall/README.md` | `--check` + `claude plugin validate` + checked-in eval suite |

Decision Record impact: aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `peer-role-mode.md` documents standalone (`gh` / chat) output and honest capability-step degradation when A2A is unmounted; `SKILL.md` routers stay within budget and declare zero `allowed-tools` (`contained` reach).
- [ ] `plugins/prompt-injection-firewall/` is registered in `PLUGIN_BUNDLES` (`displayName` without `"Anti-Sycophancy"`, `description <= 500` chars targeting untrusted issue/PR/web input, bundling `identity-firewall`, `peer-role`, and `upstream-skill-feedback`) and passes `npm run generate-claude-plugin -- --check` and `claude plugin validate ./plugins/prompt-injection-firewall`.
- [ ] Checked-in eval cases (`evals/prompt-injection-firewall/cases/*`) and the bundle's fixed `SearchPlugins` query set (`queries.json`) are committed and verified in `npm test`; live 5-session-per-arm ablation (`claude plugin eval` verified early-access gated in Claude Code `2.1.212`; standalone CLI quota window) and post-listing directory rank verification (`[L3-deferred — operator handoff needed; pre-submission 5-run ablation + post-listing SearchPlugins rank probe tracked on #446]`).
- [ ] `npm run lint && npm test` exit `0` with bumped `package.json` + `package-lock.json`.

## Out of Scope

- Modifying `AGENTS.md` L1/L2 prompt-firewall core directives or bundling `create-skill`.

## Avoided Traps

- **Putting `"Anti-Sycophancy"` in `displayName` or tuning `keywords` instead of `description` + skill names:** Deciding models read `displayName`, `description`, and skill names, and never search for sycophancy.
- **Adding hooks or `allowed-tools`:** Would downgrade Directory reach from `contained` to `privileged`.

## Related

- Parent epic: #446
- Prerequisites: #447 (`PR #452`), #453 (`PR #454`), #455 (`PR #457`), and #456 (`PR #458`)

Live latest-open + closed sweep: checked latest 20 open issues (`#289`–`#445`, `44` open total, orange band `25–50`) and open/closed search (`identity-firewall OR prompt injection plugin`, `#400`, `#443`) at `2026-11-24T21:56Z`; no equivalent found.
A2A in-flight claim sweep: latest 30 `all`-status messages checked at `2026-11-24T21:54Z`; zero competing claims.
MC sweep: `query_raw_memories` on `"plugin catalog nobody searches neomjs skills invisible standalone MCP"` (`5` hits) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "plugins/prompt-injection-firewall identity-firewall peer-role upstream-skill-feedback untrusted input OWASP ASI01"

## Timeline

- 2026-11-24T22:00:42Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T22:00:42Z @neo-gemini-julia added the `ai` label
- 2026-11-24T22:00:42Z @neo-gemini-julia added the `security` label
- 2026-11-24T22:00:43Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T22:01:18Z @neo-gemini-julia added parent issue #446
- 2026-11-24T22:03:02Z @neo-gemini-julia marked this issue as being blocked by #447
- 2026-11-24T22:25:03Z @neo-opus-grace cross-referenced by #446
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:52:36Z @neo-gemini-julia marked this issue as being blocked by #456
- 2026-11-24T23:52:38Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T01:12:29Z @neo-gemini-julia cross-referenced by PR #459

