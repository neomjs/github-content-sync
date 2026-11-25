---
id: 450
title: Add plugins/verify-before-code magnet and standalone gh spec path
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T22:00:37Z'
updatedAt: '2026-11-25T15:10:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/450'
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
  - '[x] 448 Add plugins/adversarial-pr-review magnet and standalone review path'
  - '[x] 456 Add upstream-skill-feedback skill and issue form for plugin sessions'
  - '[x] 447 Generalize generate-claude-plugin for multi-plugin marketplace'
blocking: []
closedAt: '2026-11-25T15:10:49Z'
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
- 2026-11-25T03:27:20Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T03:27:25Z @neo-gemini-julia marked this issue as being blocked by #448
### @neo-gemini-julia - 2026-11-25T03:27:27Z

## Ticket Intake, Prescription Check, & `/turn-memory-pre-flight` Record

- **Epic-review pre-check:** Parent epic #446 carries cross-family epic review `https://github.com/neomjs/neo-agent-skills/issues/446#issuecomment-7146202696` (`@neo-opus-grace`) and `issuecomment-7146287821` (`@neo-gpt-sophie`).
- **Ticket reality classification (`ticket-intake-workflow.md` §1.8):**
  - `Ticket age`: `createdAt: 2026-11-24T22:00:37Z`, `updatedAt: 2026-11-24T23:51:21Z`.
  - `Bot stale-band`: `pre-stale` (`< 7h` old; `stale: false`, `no auto close: false`).
  - `Currency / successor-risk evidence`: short-horizon drift probe (`git log origin/dev --since="2026-11-24T22:00:37Z"`) showed `#452` + `#457` merged on `origin/dev` and stacked `#458` (`v2.8.0`), `#459` (`v2.8.1`), `#460` (`v2.8.2`), and `#448` (`v2.8.3`) in flight; none deliver `plugins/verify-before-code` or standalone `gh` paths in `ticket-intake`, `ticket-create`, and `epic-review`. Sequenced as `v2.8.4` stacked on `agent/448-adversarial-pr-review` (`blocked_by #448`).
  - `ADR successor-risk`: `adr-aligned — artifact #450/2026-11-24; ADR neomjs/neo#adr:0007-agents-md-compaction-taxonomy (Accepted/2026-05-15) & neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract (Proposed/2026-05-15); evidence .agents/skills/ticket-intake/references/ticket-intake-workflow.md, .agents/skills/ticket-create/references/ticket-create-workflow.md, .agents/skills/epic-review/references/epic-review-workflow.md, scripts/lint-skill-corpus.mjs; route continue.`
  - `Substrate sufficiency audit`: existing `ticket-intake`, `ticket-create`, and `epic-review` skills assume `neo-mjs-github-workflow`, `neo-mjs-knowledge-base`, and `neo-mjs-memory-core` MCP tools without declaring a standalone `gh` CLI + `git grep`/`git log` branch or `plugins/verify-before-code/` bundle. Verdict: **`valid-as-written`** (positive ROI).
- **Stage 2 Prescription Check (`self-authored-carve.md`):**
  - `Prescription checked: .agents/skills/ticket-intake/references/ticket-intake-workflow.md — owns the concern` (pre-execution issue validation & falsification gate; standalone `gh issue view`, `gh issue list --search`, `git grep`, and `git log -S` fallback belongs here).
  - `Prescription checked: .agents/skills/ticket-create/references/ticket-create-workflow.md — owns the concern` (issue spec authoring & six-stage challenge chain; standalone `gh issue list`, `git grep`/`git log`, and user-confirmed `gh issue create` §2D rules belong here).
  - `Prescription checked: .agents/skills/epic-review/references/epic-review-workflow.md — owns the concern` (pre-work epic validation chain; standalone `gh issue view`, `gh api .../sub_issues`, `git log`, and chat/opt-in `gh issue comment` §2D rules belong here).
  - `Prescription checked: scripts/generate-claude-plugin.mjs — owns the concern` (`PLUGIN_BUNDLES` registry projecting `plugins/verify-before-code/` and `.claude-plugin/marketplace.json`).
- **`/turn-memory-pre-flight` Load-Effect Audit:**
  - Always-loaded `SKILL.md` routers (`ticket-intake/SKILL.md`, `ticket-create/SKILL.md`, `epic-review/SKILL.md`): `rewrite` (`DISCIPLINE-ONLY`) — normalize `.agents/skills/...` pointers for `${CLAUDE_SKILL_DIR}` projection and tighten verbose router prose (`<= 12` lines, net-reducing router bytes).
  - Conditional payloads (`ticket-intake-workflow.md`, `ticket-create-workflow.md`, `epic-review-workflow.md`): `rewrite` (`DISCIPLINE-ONLY`) — add compact standalone `gh` CLI + `git grep`/`git log` branches and honest capability-degradation notes while keeping total `.agents/skills/**/*.md` byte delta net-negative (`< 0 B`).

### T3 Contract Ledger Confirmation

| Target Surface | Source of Authority | Shipped Behavior | Fallback | Docs | Verification |
|---|---|---|---|---|---|
| `.agents/skills/ticket-intake/`, `ticket-create/`, `epic-review/` | `neomjs/neo#adr:0007-agents-md-compaction-taxonomy`; `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`; Directory Policy §2D/§4C | Documents standalone `gh issue`, `gh api` sub-issues, and `git grep`/`git log` sweeps + pre-code falsification, stating honestly where keyword search misses semantic KB/Memory Core prior art, and requiring user confirmation before `gh issue create`/`comment`/`edit` (§2D) | Full Agent OS MCP + A2A rules unchanged when Agent OS MCPs are mounted | `plugins/verify-before-code/README.md` | `npm run lint && npm test` |
| `plugins/verify-before-code/` & `.claude-plugin/marketplace.json` | `PLUGIN_BUNDLES` in `scripts/generate-claude-plugin.mjs`; `#446` epic review (`issuecomment-7146202696`) | Projects `ticket-intake`, `ticket-create`, `epic-review`, and `upstream-skill-feedback` (`displayName: 'GitHub Issue & Epic Pre-Code Spec Gate'`, `contained` reach, `description <= 500` chars pitched at issue pickup, spec authoring, and epic validation before coding) | Fails closed in `generate-claude-plugin.mjs --check` | `plugins/verify-before-code/README.md` | `npm run generate-claude-plugin -- --check`, `claude plugin validate`, `evals/verify-before-code/` |

- 2026-11-25T03:37:01Z @neo-gemini-julia cross-referenced by PR #462
- 2026-11-25T08:03:41Z @neo-gemini-julia referenced in commit `7c72833` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T08:59:06Z @neo-gemini-julia referenced in commit `8e1cfb2` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T10:17:22Z @neo-gemini-julia referenced in commit `477bcca` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T11:06:54Z @neo-gemini-maryam cross-referenced by PR #465
- 2026-11-25T11:10:48Z @neo-opus-ada marked this issue as blocking #463
- 2026-11-25T11:21:27Z @neo-opus-ada removed the block on #463
- 2026-11-25T11:24:04Z @neo-gemini-julia referenced in commit `087f5fe` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T11:30:22Z @neo-gemini-julia referenced in commit `5a3a114` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T11:54:43Z @neo-gemini-julia referenced in commit `801c174` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T12:00:12Z @neo-gemini-julia referenced in commit `de65b3d` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T12:54:02Z @neo-gemini-julia referenced in commit `7df8338` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T13:21:26Z @neo-gemini-julia referenced in commit `6afa670` - "feat(claude): add verify-before-code plugin bundle (#450)"
- 2026-11-25T14:46:36Z @neo-gemini-julia referenced in commit `36b46f9` - "fix(skills): address Round-1 review feedback on verify-before-code bundle (#450)"
- 2026-11-25T15:10:49Z @tobiu referenced in commit `8f1e8f6` - "feat(claude): add verify-before-code plugin bundle (#450) (#462)

* feat(claude): add verify-before-code plugin bundle (#450)

* fix(skills): address Round-1 review feedback on verify-before-code bundle (#450)"
- 2026-11-25T15:10:50Z @tobiu closed this issue

