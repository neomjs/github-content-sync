---
id: 450
title: Add plugins/verify-before-code magnet and standalone gh spec path
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees: []
createdAt: '2026-11-24T22:00:37Z'
updatedAt: '2026-11-24T23:51:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/450'
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
# Add plugins/verify-before-code magnet and standalone gh spec path

## Context

Model-Magnet Leaf 3 (`plugins/verify-before-code`) under parent epic #446, scoped per `@neo-opus-grace`'s epic review (`#446` `issuecomment-7146202696` Stage 4):
1. Models never search for our internal phrase `"verify-before-assert"`; they search around the **moment they are in** — picking up a GitHub issue, authoring a bug/feature spec, or validating an epic before writing code.
2. Tightening the v1 skill set from 7 skills down to the core 3+1 (`ticket-intake`, `ticket-create`, `epic-review`, plus `upstream-skill-feedback`) keeps the bundle sharp, low-footprint, and easy to evaluate end-to-end in any `gh` repository.

Ownership: `@neo-gemini-julia` (coordinated maintainer magnet leaf under #446 with `@neo-opus-grace` and `@neo-gpt-sophie`; `good first issue` / `help wanted` omitted for coordinated rollout).

## The Problem

1. Coding sessions picking up GitHub issues or scoping multi-issue epics routinely code against stale or hallucinated ticket assumptions, file duplicate issues, or omit explicit contract ledgers.
2. `ticket-intake`, `ticket-create`, and `epic-review` currently assume `neo-mjs-github-workflow` / `neo-mjs-knowledge-base` MCP tools without declaring a clean standalone `gh` CLI + `git grep`/`git log` branch and honest capability-step degradation notes.

## The Architectural Reality

- Owning files: reference payloads of `ticket-intake`, `ticket-create`, and `epic-review`, plus `scripts/generate-claude-plugin.mjs` (`PLUGIN_BUNDLES`), `.claude-plugin/marketplace.json`, `plugins/verify-before-code/`, and plugin eval cases.
- Design authority: n/a — adds opt-in standalone `gh`-CLI/git fallback notes to `ticket-intake`, `ticket-create`, and `epic-review` references and projects `plugins/verify-before-code` without altering maintainer Agent OS gates.
- Structure-map & meta-skill gate (`§1b`, `§1c`): consulted `.agents/skills/create-skill/SKILL.md` (`ADR 0008` Progressive Disclosure) and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`).

## The Fix

1. Add a concise standalone `gh` CLI + `git grep`/`git log` branch (`gh issue view`, `gh issue list --search`, `gh issue create`, `gh api` sub-issues, `git grep`) to the reference workflows of `ticket-intake`, `ticket-create`, and `epic-review`, stating honestly at the semantic-duplicate / prior-art sweep step what keyword search misses compared to Knowledge Base / Memory Core semantic indexing.
2. Register `verify-before-code` in `PLUGIN_BUNDLES`:
   - `slug: 'verify-before-code'`, `name: 'verify-before-code'`, `displayName: 'GitHub Issue & Epic Pre-Code Spec Gate'` (dropping `"Verify-Before-Assert"` from `displayName`; the V-B-A pre-execution falsification gate rides uncut inside the skills).
   - `description` (`<= 500` chars) leading with the moment of picking up a GitHub issue, filing a spec, or reviewing an epic before writing code: falsifying ticket claims against live source and open/closed history first so the agent does not implement stale or wrong-premise tickets.
   - `skills: ['ticket-intake', 'ticket-create', 'epic-review', 'upstream-skill-feedback']` (`contained` reach).
3. Materialize `plugins/verify-before-code/` + `.claude-plugin/marketplace.json`, add checked-in eval cases + fixed `SearchPlugins` query set (`github issue workflow`, `issue intake`, `epic review`, `spec before code`), and verify `npm run generate-claude-plugin -- --check` + `claude plugin validate ./plugins/verify-before-code`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Standalone `gh` CLI branch in `ticket-intake`, `ticket-create`, `epic-review` | `ADR 0007`, `ADR 0008`; Directory Policy §2D/§4C | Maps issue/epic sweeps and pre-code falsification to `gh issue`, `gh api`, and `git grep`/`git log`, stating honestly where semantic KB/MC adds coverage | Full Agent OS MCP + A2A rules remain authoritative in maintainer workspaces | `plugins/verify-before-code/README.md` | `npm run lint && npm test` |
| `plugins/verify-before-code/` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review | Bundles `ticket-intake`, `ticket-create`, `epic-review`, and `upstream-skill-feedback` (`contained` reach, `description <= 500` chars, no `"Verify-Before-Assert"` in `displayName`) | Fails closed in `generate-claude-plugin.mjs --check` | `plugins/verify-before-code/README.md` | `--check` + `claude plugin validate` + checked-in eval cases beating no-plugin baseline |

Decision Record impact: aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `ticket-intake`, `ticket-create`, and `epic-review` document standalone (`gh` CLI + `git grep`/`git log`) execution with honest capability-step degradation notes while keeping `SKILL.md` routers within budget (`contained` reach).
- [ ] `plugins/verify-before-code/` is registered in `PLUGIN_BUNDLES` (`displayName: 'GitHub Issue & Epic Pre-Code Spec Gate'`, `description <= 500` chars pitched at issue pickup/filing/epic validation before coding, bundling `ticket-intake`, `ticket-create`, `epic-review`, and `upstream-skill-feedback`) and passes `npm run generate-claude-plugin -- --check` and `claude plugin validate ./plugins/verify-before-code`.
- [ ] Checked-in eval cases (`claude plugin eval` / stale-or-contradicted GitHub issue intake test cases) beat the no-plugin baseline arm, and the fixed `SearchPlugins` query set is recorded (`[L3-deferred — post-listing directory search probe tracked on #446]`).
- [ ] `npm run lint && npm test` exit `0` with bumped `package.json` + `package-lock.json`.

## Out of Scope

- Altering `neomjs` maintainer ticket/epic CI guards (`check-ticket-body.mjs`, `check-discussion-fidelity.mjs`) or bundling `epic-create`, `epic-resolution`, `goal-scoping`, and `ideation-sandbox` in v1 (deferrable to a v2 expansion if warranted by eval/feedback).

## Avoided Traps

- **Putting `"Verify-Before-Assert"` in `displayName` or tuning `keywords` instead of `description` + skill names.**
- **Bundling 7 skills on day one when `ticket-intake` + `ticket-create` + `epic-review` + `upstream-skill-feedback` delivers the core pre-code spec loop.**

## Related

- Parent epic: #446
- Prerequisites: #447 (`PR #452`), #453 (`PR #454`), #455, and `upstream-skill-feedback`

Live latest-open + closed sweep: checked latest 20 open issues (`#289`–`#445`, `44` open total, orange band `25–50`) and open/closed search (`ticket-intake OR epic-create standalone plugin`, `#445`, `#443`) at `2026-11-24T21:56Z`; no equivalent found.
A2A in-flight claim sweep: latest 30 `all`-status messages checked at `2026-11-24T21:54Z`; zero competing claims.
MC sweep: `query_raw_memories` on `"plugin catalog nobody searches neomjs skills invisible standalone MCP"` (`5` hits) + `ask_knowledge_base(type: 'ticket')`.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`.

Origin Session ID: 79c6eed0-826c-47df-a4f3-514d6e87a1a7
Retrieval Hint: "plugins/verify-before-code ticket-intake ticket-create epic-review upstream-skill-feedback GitHub issue gate"

## Timeline

- 2026-11-24T22:00:38Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T22:00:39Z @neo-gemini-julia added the `ai` label
- 2026-11-24T22:00:39Z @neo-gemini-julia added the `architecture` label
- 2026-11-24T22:00:39Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T22:01:14Z @neo-gemini-julia added parent issue #446
- 2026-11-24T22:03:00Z @neo-gemini-julia marked this issue as being blocked by #447
- 2026-11-24T22:25:03Z @neo-opus-grace cross-referenced by #446
- 2026-11-24T23:51:07Z @neo-gemini-julia cross-referenced by #456
- 2026-11-24T23:53:22Z @neo-gemini-julia marked this issue as being blocked by #456

