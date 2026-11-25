---
id: 447
title: Generalize generate-claude-plugin for multi-plugin marketplace
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - model-experience
assignees:
  - neo-opus-grace
createdAt: '2026-11-24T21:58:45Z'
updatedAt: '2026-11-24T23:00:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/447'
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
blockedBy: []
blocking:
  - '[ ] 451 Add plugins/prompt-injection-firewall magnet for untrusted input'
  - '[ ] 450 Add plugins/verify-before-code magnet and standalone gh spec path'
  - '[ ] 449 Add plugins/context-compaction-memory magnet and standalone fallbacks'
  - '[ ] 448 Add plugins/adversarial-pr-review magnet and standalone review path'
closedAt: '2026-11-24T23:00:59Z'
---
# Generalize generate-claude-plugin for multi-plugin marketplace

## Context

Foundation leaf for parent epic #446 (`Model-magnet Claude plugin suite and multi-plugin marketplace`). Currently, `scripts/generate-claude-plugin.mjs` (`#443` / `PR #444`) hardcodes a single output bundle (`PLUGIN_REL = 'plugins/neo'`) derived solely from `audiences: ["app"]`, emits no root `.claude-plugin/marketplace.json` catalog for `claude plugin marketplace add neomjs/neo-agent-skills`, and indexes `plugins/neo/.claude-plugin/plugin.json` around internal proper nouns (`"Neo.mjs"`, `"Neural Link"`) rather than the complex UI capability tokens (`data-grid`, `gantt`, `scheduler`, `kanban`, `multi-window`, `docking`, `web-desktop`, `offscreen-canvas`, `live-dom-inspection`) that model sessions actually query when tasked with building rich applications.

`handoff: @neo-opus-grace` (coordinated maintainer foundation leaf with `@neo-gemini-julia` and `@neo-opus-grace`; `good first issue` / `help wanted` omitted because sibling magnet bundles depend on `PLUGIN_BUNDLES`).

## The Problem

1. `scripts/generate-claude-plugin.mjs` cannot project multiple targeted `plugins/<slug>/` bundles or a root `.claude-plugin/marketplace.json` catalog.
2. When a coding-agent session is asked to build a high-frequency data grid, Gantt/Scheduler, Kanban board, or tear-out multi-window desktop workspace, its marketplace search queries capability tokens (`"data grid"`, `"gantt"`, `"scheduler"`, `"multi-window"`, `"docking"`) — missing `plugins/neo` completely because `expectedPluginManifest()` omits those capability nouns and does not explain to the model why persistent Web Worker / SharedWorker JSON component graphs + `neo-neural-link` prevent blind reload/screenshot loops.

## The Architectural Reality

- Owning files: `scripts/generate-claude-plugin.mjs`, `scripts/test-claude-plugin.mjs`, `.claude-plugin/marketplace.json`, and `plugins/neo/`.
- Design authority: n/a — generalizes the opt-in Claude plugin generator introduced in `#443` to a declarative multi-bundle table + root `marketplace.json` without altering `.agents/skills/**` or npm package files.
- Each bundle under `plugins/<slug>/` must remain independently valid under both `claude plugin validate ./plugins/<slug>` and Anthropic's `claude.ai/directory/manage` portal scanner (`1024x1024` `assets/icon.png`, real files, zero symlinks, zero `stdio` `.mcp.json`, `${CLAUDE_SKILL_DIR}` / `${CLAUDE_PLUGIN_ROOT}` / HTTPS blob path rewriting, and `version` synced to root `package.json`).
- Structure-map & Structural Pre-Flight (`§1c`): ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`); modifies existing `scripts/generate-claude-plugin.mjs` and `scripts/test-claude-plugin.mjs` in place.

## The Fix

1. Refactor `scripts/generate-claude-plugin.mjs` around an exported `PLUGIN_BUNDLES` registry (starting with `'neo'` and ready for sibling magnet entries in `plugins/<slug>/`), where each bundle declares `{slug, name, displayName, description, keywords, skills, mcpSetup}` and receives its own `.claude-plugin/plugin.json`, `assets/icon.png` (copied from canonical `plugins/neo/assets/icon.png`), `README.md`, and path-rewritten `skills/<skill>/` tree.
2. Emit and `--check`-verify `.claude-plugin/marketplace.json` at the repository root listing every registered bundle in `PLUGIN_BUNDLES` (`source: "./plugins/<slug>"`, `version` synced to `package.json`), and verify `claude plugin validate .` / `claude plugin validate ./plugins/neo`.
3. Rewrite `plugins/neo`'s `description`, `keywords`, and `README.md` model-to-model around complex UI & runtime inspection capabilities (`data-grid`, `virtualized-table`, `gantt`, `scheduler`, `kanban`, `multi-window`, `docking`, `web-desktop`, `web-workers`, `sharedworker`, `offscreen-canvas`, `live-dom-inspection`, `context-recovery`) so models discover why `"The Application Engine for the AI Era"` eliminates blind main-thread DOM reload loops.
4. Extend `scripts/test-claude-plugin.mjs` to enforce multi-bundle parity, `.claude-plugin/marketplace.json` sync, per-bundle `${CLAUDE_SKILL_DIR}` / `${CLAUDE_PLUGIN_ROOT}` path rewriting, and `npm version` lifecycle sync across all registered bundles.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `PLUGIN_BUNDLES` & `.claude-plugin/marketplace.json` (`scripts/generate-claude-plugin.mjs`) | Claude Code multi-plugin `marketplace.json` spec & `claude.ai/directory/manage` scanner (`#443`) | Materializes and `--check`-verifies every registered `plugins/<slug>/` bundle plus root `.claude-plugin/marketplace.json` synced to `package.json` `version` | Fails closed (`exit 1`) in `npm test` on any drift, symlink, `.mcp.json`, or unrewritten `.agents/skills/` path | `plugins/neo/README.md` | `node scripts/test-claude-plugin.mjs` + `claude plugin validate` |
| `plugins/neo/.claude-plugin/plugin.json` | `neomjs/neo` `learn/benefits/Introduction.md` | Indexes `plugins/neo` for model capability searches (`data-grid`, `gantt`, `scheduler`, `kanban`, `multi-window`, `docking`, `web-desktop`, `offscreen-canvas`, `live-dom-inspection`, `context-recovery`) | Preserves `name: "neomjs"` & `displayName: "Neo.mjs"` continuity for the existing Claude Directory submission | `plugins/neo/README.md` | `node scripts/test-claude-plugin.mjs` + `claude plugin validate ./plugins/neo` |

Decision Record impact: aligned-with `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `scripts/generate-claude-plugin.mjs` exports declarative `PLUGIN_BUNDLES` and materializes + `--check`-verifies both `plugins/<slug>/` bundles and root `.claude-plugin/marketplace.json` (kept out of npm `"files"` so the `neo-agent-skills` npm tarball stays lean).
- [ ] `plugins/neo/.claude-plugin/plugin.json` and `plugins/neo/README.md` carry model-targeted capability indexing (`data-grid`, `gantt`, `scheduler`, `kanban`, `multi-window`, `docking`, `web-desktop`, `web-workers`, `sharedworker`, `offscreen-canvas`, `live-dom-inspection`, `context-recovery`) while retaining `name: "neomjs"` and `version` synced to `package.json`.
- [ ] `scripts/test-claude-plugin.mjs` verifies multi-bundle projection, `.claude-plugin/marketplace.json` parity, cross-bundle `${CLAUDE_PLUGIN_ROOT}` vs. HTTPS URL path rewriting, and `claude plugin validate` passes with zero blocking errors.
- [ ] `.agents/skills/**` grows by `0 B`, `package.json` + `package-lock.json` bump cleanly, and `npm run lint && npm test` exit `0`.

## Out of Scope

- Adding standalone `gh`-CLI fallback branches to maintainer skills (owned by the four sibling magnet leaves under #446).

## Avoided Traps

- **Including `.claude-plugin` or `plugins/` in root `package.json` `"files"`:** Would bloat every `npm install neo-agent-skills` tarball with PNG icons and duplicated markdown; keep plugins Git-resident for Claude Directory & GitHub marketplace installs.

## Related

- Parent epic: #446
- Precedent: `#443` (`PR #444`)

Live latest-open + closed sweep: checked latest 20 open issues (`#289`–`#445`, `44` open total, orange band `25–50`) and open/closed search (`plugin OR marketplace`, `#443`, `#387`, `#287`) at `2026-11-24T21:56Z`; no equivalent found.
A2A in-flight claim sweep: latest 30 `all`-status messages checked at `2026-11-24T21:54Z`; zero competing claims.
MC sweep: `query_raw_memories` on `"plugin catalog nobody searches neomjs skills invisible standalone MCP"` (`5` hits) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "PLUGIN_BUNDLES marketplace.json generate-claude-plugin multi-plugin UI capability keywords"

## Timeline

- 2026-11-24T21:58:46Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T21:58:46Z @neo-gemini-julia added the `ai` label
- 2026-11-24T21:58:47Z @neo-gemini-julia added the `build` label
- 2026-11-24T21:58:47Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T22:00:33Z @neo-gemini-julia cross-referenced by #448
- 2026-11-24T22:00:36Z @neo-gemini-julia cross-referenced by #449
- 2026-11-24T22:00:38Z @neo-gemini-julia cross-referenced by #450
- 2026-11-24T22:00:41Z @neo-gemini-julia cross-referenced by #451
- 2026-11-24T22:01:05Z @neo-gemini-julia added parent issue #446
- 2026-11-24T22:02:54Z @neo-gemini-julia marked this issue as blocking #448
- 2026-11-24T22:02:57Z @neo-gemini-julia marked this issue as blocking #449
- 2026-11-24T22:03:00Z @neo-gemini-julia marked this issue as blocking #450
- 2026-11-24T22:03:02Z @neo-gemini-julia marked this issue as blocking #451
- 2026-11-24T22:25:03Z @neo-opus-grace cross-referenced by #446
- 2026-11-24T22:25:12Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-24T22:36:09Z @neo-opus-grace cross-referenced by PR #452
- 2026-11-24T23:00:59Z @tobiu referenced in commit `9039405` - "feat(claude): project every plugin bundle and a root marketplace from one registry (#447) (#452)

PLUGIN_BUNDLES is harness-neutral (slug, name, description, skills or audience); the Claude projector writes each plugins/<slug>/ bundle and .claude-plugin/marketplace.json, and --check fails closed on registry, parity, marketplace and orphan-bundle drift. plugins/neo's description now names the UI work it answers. 2.7.0."
- 2026-11-24T23:01:00Z @tobiu closed this issue
- 2026-11-24T23:08:40Z @neo-opus-grace cross-referenced by #453
- 2026-11-24T23:42:39Z @neo-gemini-julia cross-referenced by PR #454
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456

