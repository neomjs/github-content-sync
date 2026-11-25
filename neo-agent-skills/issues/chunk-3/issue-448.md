---
id: 448
title: Add plugins/adversarial-pr-review magnet and standalone review path
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T22:00:32Z'
updatedAt: '2026-11-25T13:12:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/448'
author: neo-gemini-julia
commentsCount: 1
parentIssue: 446
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 449 Add plugins/context-compaction-memory magnet and standalone fallbacks'
  - '[x] 456 Add upstream-skill-feedback skill and issue form for plugin sessions'
  - '[x] 447 Generalize generate-claude-plugin for multi-plugin marketplace'
blocking:
  - '[x] 450 Add plugins/verify-before-code magnet and standalone gh spec path'
closedAt: '2026-11-25T13:12:36Z'
---
# Add plugins/adversarial-pr-review magnet and standalone review path

## Context

Model-Magnet Leaf 1 (`plugins/adversarial-pr-review`) under parent epic #446 — sequenced after `upstream-skill-feedback`, `#455`, `#451`, and `#449` (`#446` `issuecomment-7146202696` Stage 4). Because Anthropic's first-party `code-review` and `pr-review-toolkit` already top generic `"code review"` searches, this bundle differentiates in its `description` (`<= 500` chars) on what first-party review plugins lack: **patch-blind review against the linked ticket's acceptance criteria and premise (*"can the handled state exist in practice?"*), since green CI does not prove ACs are met**.

Ownership: `@neo-gemini-julia` (coordinated maintainer magnet leaf under #446 with `@neo-opus-grace` and `@neo-gpt-sophie`; `good first issue` / `help wanted` omitted for coordinated rollout).

## The Problem

1. While `.agents/skills/pull-request/` already carries a standalone `contributor` (`gh` CLI) path, `.agents/skills/pr-review/` and `.agents/skills/tech-debt-radar/` assume internal Agent OS MCP tools (`manage_pr_review`, `get_pull_request_diff`, `query_raw_memories`, `ask_knowledge_base`, `AGENT:*` A2A) without a standalone `gh` CLI + `git log` branch and honest capability-step degradation note.
2. Without `plugins/adversarial-pr-review` carrying a sharp `<= 500`-char `description` and bundled `upstream-skill-feedback`, external sessions cannot run our Patch-Blind Premise Snapshot, Evidence Audit, and `can-this-state-exist` checks in arbitrary `gh` repositories.

## The Architectural Reality

- Owning files: `.agents/skills/pr-review/references/pr-review-guide.md`, `.agents/skills/tech-debt-radar/`, `scripts/generate-claude-plugin.mjs` (`PLUGIN_BUNDLES`), `.claude-plugin/marketplace.json`, `plugins/adversarial-pr-review/`, and plugin eval cases.
- Design authority: `"Contributors use §§1, 3–7, 12–13 with public issue/PR history, commits and fresh source as their intent record."` (`.agents/skills/pull-request/references/review-response-protocol.md` lines 4–5).
- Budget constraints (`skills.manifest.json` & `ADR 0007` / `ADR 0008`): `pr-review` loaded surface must stay well under its `41,357 B` `COMBINED_BUDGETS` ceiling (`35,250 B` today), `SKILL.md` routers stay `<= 12` lines / `<= 220 B` description, and the bundle stays `contained` (zero `allowed-tools`, hooks, or `.mcp.json`).
- Structure-map & meta-skill gate (`§1b`, `§1c`): consulted `.agents/skills/create-skill/SKILL.md` (`ADR 0008`) and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`).

## The Fix

1. Add a compact standalone `gh` CLI + `git log` execution branch (`gh pr view`, `gh pr diff`, `gh pr checks`, `gh pr review`, `gh issue view`, `git log -S`) to `pr-review/references/pr-review-guide.md` and `tech-debt-radar`, stating honestly at the prior-art / semantic-sweep step what `git log` + `gh search` cannot recover compared to cross-session Memory Core / Knowledge Base.
2. Register `adversarial-pr-review` in `PLUGIN_BUNDLES`:
   - `description` (`<= 500` chars) leading with patch-blind PR review against the ticket's acceptance criteria, Contract Ledger, and real-state premise (*"can the handled state exist in practice?"*) when green CI still lets wrong-premise or incomplete PRs slip through.
   - `skills: ['pr-review', 'pull-request', 'tech-debt-radar', 'upstream-skill-feedback']` (`contained` reach).
3. Materialize `plugins/adversarial-pr-review/` + `.claude-plugin/marketplace.json`, check in eval cases + fixed `SearchPlugins` query set, and verify `npm run generate-claude-plugin -- --check` + `claude plugin validate ./plugins/adversarial-pr-review`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `pr-review` & `tech-debt-radar` standalone path | `review-response-protocol.md` audience split; `ADR 0007` / `ADR 0008`; Directory Policy §2D/§4C | Runs Patch-Blind Premise Snapshot, AC/Contract-Ledger audit, and debt sweeps via `gh` CLI + `git log`, with honest step-level capability notes where Memory Core/KB is absent | Full Agent OS maintainer rules unchanged when Agent OS MCPs are mounted | `plugins/adversarial-pr-review/README.md` | `npm run lint && npm test` |
| `plugins/adversarial-pr-review/` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review | Bundles `pr-review`, `pull-request`, `tech-debt-radar`, and `upstream-skill-feedback` (`contained` reach, `description <= 500` chars) | Fails closed in `generate-claude-plugin.mjs --check` | `plugins/adversarial-pr-review/README.md` | `--check` + `claude plugin validate` + checked-in eval cases beating no-plugin baseline |

Decision Record impact: aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `pr-review` and `tech-debt-radar` document standalone (`gh` CLI + `git log`) execution with honest capability-step degradation notes, keeping `pr-review` below its `41,357 B` cap and `SKILL.md` routers within budget (`contained` reach).
- [ ] `plugins/adversarial-pr-review/` is registered in `PLUGIN_BUNDLES` (`description <= 500` chars differentiated on ticket-AC and real-premise verification beyond green CI, bundling `pr-review`, `pull-request`, `tech-debt-radar`, and `upstream-skill-feedback`) and passes `npm run generate-claude-plugin -- --check` and `claude plugin validate ./plugins/adversarial-pr-review`.
- [ ] Checked-in eval cases (`claude plugin eval` / seeded wrong-premise green-test PR diff cases) beat the no-plugin baseline arm, and the fixed `SearchPlugins` query set is recorded (`[L3-deferred — post-listing directory search probe tracked on #446]`).
- [ ] `npm run lint && npm test` exit `0` with bumped `package.json` + `package-lock.json`.

## Out of Scope

- Weakening any `neomjs` maintainer cross-family review gate or `review-budget-managed` CI enforcement.

## Avoided Traps

- **Relying on `keywords` instead of `description` + skill names, or competing head-on with generic style/lint review copy.**
- **Breaching `pr-review`'s `41,357 B` `COMBINED_BUDGETS` ceiling.**

## Related

- Parent epic: #446
- Prerequisites: #447 (`PR #452`), #453 (`PR #454`), #455, and `upstream-skill-feedback`

Live latest-open + closed sweep: checked latest 20 open issues (`#289`–`#445`, `44` open total, orange band `25–50`) and open/closed search (`pr-review standalone plugin`, `#432`, `#443`) at `2026-11-24T21:56Z`; no equivalent found.
A2A in-flight claim sweep: latest 30 `all`-status messages checked at `2026-11-24T21:54Z`; zero competing claims.
MC sweep: `query_raw_memories` on `"plugin catalog nobody searches neomjs skills invisible standalone MCP"` (`5` hits) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "plugins/adversarial-pr-review pr-review pull-request tech-debt-radar upstream-skill-feedback premise AC audit"

## Timeline

- 2026-11-24T22:00:34Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T22:00:34Z @neo-gemini-julia added the `ai` label
- 2026-11-24T22:00:35Z @neo-gemini-julia added the `architecture` label
- 2026-11-24T22:00:35Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T22:01:08Z @neo-gemini-julia added parent issue #446
- 2026-11-24T22:02:54Z @neo-gemini-julia marked this issue as being blocked by #447
- 2026-11-24T22:25:03Z @neo-opus-grace cross-referenced by #446
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:53:16Z @neo-gemini-julia marked this issue as being blocked by #456
- 2026-11-25T03:17:53Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T03:17:59Z @neo-gemini-julia marked this issue as being blocked by #449
### @neo-gemini-julia - 2026-11-25T03:18:00Z

## Ticket Intake, Prescription Check, & `/turn-memory-pre-flight` Record

- **Epic-review pre-check:** Parent epic #446 carries cross-family epic review `https://github.com/neomjs/neo-agent-skills/issues/446#issuecomment-7146202696` (`@neo-opus-grace`) and `issuecomment-7146287821` (`@neo-gpt-sophie`).
- **Ticket reality classification (`ticket-intake-workflow.md` §1.8):**
  - `Ticket age`: `createdAt: 2026-11-24T22:00:32Z`, `updatedAt: 2026-11-24T23:51:18Z`.
  - `Bot stale-band`: `pre-stale` (`< 7h` old; `stale: false`, `no auto close: false`).
  - `Currency / successor-risk evidence`: short-horizon drift probe (`git log origin/dev --since="2026-11-24T22:00:32Z"`) showed `#452` + `#457` merged on `origin/dev` and stacked `#458` (`v2.8.0`), `#459` (`v2.8.1`), and `#460` (`v2.8.2`) in flight; none deliver `plugins/adversarial-pr-review` or standalone `gh` paths in `pr-review`/`tech-debt-radar`. Sequenced as `v2.8.3` stacked on `agent/449-context-compaction-memory` (`blocked_by #449`).
  - `ADR successor-risk`: `adr-aligned — artifact #448/2026-11-24; ADR neomjs/neo#adr:0007-agents-md-compaction-taxonomy (Accepted/2026-05-15) & neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract (Proposed/2026-05-15); evidence .agents/skills/pr-review/references/pr-review-guide.md, .agents/skills/tech-debt-radar/references/tech-debt-radar-guide.md, scripts/lint-skill-corpus.mjs; route continue.`
  - `Substrate sufficiency audit`: existing `pr-review` and `tech-debt-radar` skills assume internal Agent OS MCP tools (`manage_pr_review`, `get_pull_request_diff`, `ask_knowledge_base`, `query_raw_memories`, `AGENT:*` A2A) without a standalone `gh` CLI + `git log` execution branch or `plugins/adversarial-pr-review/` bundle. Verdict: **`valid-as-written`** (positive ROI).
- **Stage 2 Prescription Check (`self-authored-carve.md`):**
  - `Prescription checked: .agents/skills/pr-review/references/pr-review-guide.md — owns the concern` (reviewer workflow & Patch-Blind Premise Snapshot; standalone `gh` CLI + `git log -S` branch belongs in the reference guide while keeping the `COMBINED_BUDGETS` loaded surface `pr-review-guide.md` + `review-cost-circuit-breaker.md` strictly below `41,357 B` and net-reducing `.agents/skills/**/*.md`).
  - `Prescription checked: .agents/skills/tech-debt-radar/references/tech-debt-radar-guide.md — owns the concern` (ambient debt sweep method; standalone `gh issue/pr list --search` + `git log -S` + `wc -l` fallback and honest semantic-RAG capability note belong in the reference guide).
  - `Prescription checked: scripts/generate-claude-plugin.mjs — owns the concern` (`PLUGIN_BUNDLES` registry projecting `plugins/adversarial-pr-review/` and `.claude-plugin/marketplace.json`).
- **`/turn-memory-pre-flight` Load-Effect Audit:**
  - Always-loaded `SKILL.md` routers (`pr-review/SKILL.md`, `tech-debt-radar/SKILL.md`): `rewrite` — normalize repo-relative `.agents/skills/...` pointers for `${CLAUDE_SKILL_DIR}` projection and tighten verbose router prose (`<= 0 B` router delta, `<= 11` lines).
  - Conditional payloads (`pr-review/references/pr-review-guide.md`, `tech-debt-radar/references/tech-debt-radar-guide.md`): `move`/`rewrite` — add compact standalone `gh` CLI + `git log` execution branches and honest capability-step degradation notes (`DISCIPLINE-ONLY`), offset by trimming redundant prose so total `.agents/skills/**/*.md` byte delta is net-negative (`< 0 B`).

### T3 Contract Ledger Confirmation

| Target Surface | Source of Authority | Shipped Behavior | Fallback | Docs | Verification |
|---|---|---|---|---|---|
| `.agents/skills/pr-review/` (`SKILL.md`, `references/pr-review-guide.md`) | `review-response-protocol.md` audience split; `neomjs/neo#adr:0007-agents-md-compaction-taxonomy`; `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`; Directory Policy §2D/§4C | Documents standalone `gh` CLI + `git log -S` Patch-Blind Premise Snapshot, AC/Contract-Ledger audit, and review submission (`gh pr view`, `gh pr checks`, `gh pr diff`, `gh pr review` or chat output), stating honestly what `git log` + `gh` keyword search cannot recover vs. cross-session Memory Core / Knowledge Base; preserves `COMBINED_BUDGETS < 41,357 B` | Full Agent OS maintainer MCP + cross-family review-budget rules unchanged when Agent OS MCPs are mounted | `plugins/adversarial-pr-review/README.md` | `npm run lint && npm test` |
| `.agents/skills/tech-debt-radar/` (`SKILL.md`, `references/tech-debt-radar-guide.md`) | `neomjs/neo#adr:0007-agents-md-compaction-taxonomy`; `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`; Directory Policy §4C | Documents standalone `gh issue/pr list --search`, `git log -S`/`--grep`, local `docs/adr` + `ARCHITECTURE.md` authority derivation, and `find`/`wc -l` outlier scans, with honest semantic-RAG capability boundary | Full Agent OS KB + Memory Core semantic RAG sweep unchanged when mounted | `plugins/adversarial-pr-review/README.md` | `npm run lint && npm test` |
| `plugins/adversarial-pr-review/` & `.claude-plugin/marketplace.json` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review (`issuecomment-7146202696`) | Projects `pr-review`, `pull-request`, `tech-debt-radar`, and `upstream-skill-feedback` (`contained` reach, `description <= 500` chars differentiated on patch-blind ticket-AC and real-state premise verification beyond green CI) | Fails closed in `generate-claude-plugin.mjs --check` | `plugins/adversarial-pr-review/README.md` | `npm run generate-claude-plugin -- --check`, `claude plugin validate`, `evals/adversarial-pr-review/` |

- 2026-11-25T03:27:15Z @neo-gemini-julia cross-referenced by PR #461
- 2026-11-25T03:27:25Z @neo-gemini-julia marked this issue as blocking #450
- 2026-11-25T10:18:29Z @neo-gemini-julia cross-referenced by PR #462
- 2026-11-25T11:51:39Z @neo-gemini-julia referenced in commit `03f5812` - "feat(claude): add adversarial-pr-review plugin bundle (#448)"
- 2026-11-25T12:00:08Z @neo-gemini-julia referenced in commit `d5107fb` - "feat(claude): add adversarial-pr-review plugin bundle (#448)"
- 2026-11-25T12:48:57Z @neo-gemini-julia referenced in commit `402ee97` - "fix(claude): split router audiences, fix grep -E, and add discriminating eval (#448)"
- 2026-11-25T13:12:36Z @tobiu referenced in commit `ff7f311` - "feat(claude): add adversarial-pr-review plugin bundle (#448) (#461)

* feat(claude): add adversarial-pr-review plugin bundle (#448)

* fix(claude): split router audiences, fix grep -E, and add discriminating eval (#448)"
- 2026-11-25T13:12:38Z @tobiu closed this issue

