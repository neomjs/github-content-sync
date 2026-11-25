---
id: 456
title: Add upstream-skill-feedback skill and issue form for plugin sessions
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-24T23:51:05Z'
updatedAt: '2026-11-25T01:03:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/456'
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
  - '[ ] 450 Add plugins/verify-before-code magnet and standalone gh spec path'
  - '[ ] 449 Add plugins/context-compaction-memory magnet and standalone fallbacks'
  - '[ ] 448 Add plugins/adversarial-pr-review magnet and standalone review path'
  - '[ ] 451 Add plugins/prompt-injection-firewall magnet for untrusted input'
---
# Add upstream-skill-feedback skill and issue form for plugin sessions

## Context

Sub-ticket under parent epic #446 (`Model-magnet Claude plugin suite and multi-plugin marketplace`), surfaced by `@neo-opus-grace`'s epic review (`issuecomment-7146202696`, Stage 3.1) and directory-policy audit (`issuecomment-7146635341`). When an external Claude, Codex, or Antigravity session installs one of our projected plugin bundles (`plugins/neo` or `#448`–`#451`) and a bundled skill misfires, hits an unmapped standalone step, or encounters a gap, nothing turns that live external friction into a structured upstream issue — and `neomjs/neo-agent-skills` currently ships zero `.github/ISSUE_TEMPLATE/` definitions.

Ownership: self-assigned (`@neo-gemini-julia`, coordinated rollout under #446; `good first issue` / `help wanted` omitted because `#448`–`#451` depend on bundling this skill).

## The Problem

1. External agent sessions using our plugin bundles outside an Agent OS maintainer workspace have no opt-in `friction -> gold` path back to `neomjs/neo-agent-skills` when a skill assumption fails in their repository.
2. Without a structured GitHub Issue Form (`.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml`) and explicit consent + redaction + user-actor submission rules, external reports either never happen, risk leaking private workspace details, or risk violating Anthropic Software Directory Policy §2D (`"must not intentionally call or coerce Claude into calling other external software"`) and §4C (never a promotional vehicle).

## The Architectural Reality

- Owning files: `.agents/skills/upstream-skill-feedback/SKILL.md`, `.agents/skills/upstream-skill-feedback/references/upstream-feedback-workflow.md`, `.agents/skills/skills.manifest.json`, `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml`, `agents-md/sections/0902-app-skills.md`, `agents-md/sections/0903-contributor-skills.md`, `scripts/generate-claude-plugin.mjs` (`plugins/neo` via `audience: 'app'` + magnet bundles), and `plugins/neo/` + `.claude-plugin/marketplace.json`.
- Design authority: n/a — adds a new opt-in `upstream-skill-feedback` skill and GitHub Issue Form under epic #446 without changing existing skill behavior or adding a competing filing path in maintainer sessions (`ticket-create` / `defect-note` remain the maintainer path).
- Directory Policy & Reach constraints (`issuecomment-7146635341`, `PR #458` review):
  - Zero `allowed-tools` frontmatter, zero hooks, zero MCP servers — keeps every bundle's Directory reach `contained`.
  - User stays the default actor for the outward write (§2D): after redaction + consent preview, the skill outputs a pre-filled GitHub Issue Form link (`.../issues/new?template=plugin-skill-feedback.yml&...`) for the user to open and submit, running `gh issue create --repo neomjs/neo-agent-skills` only when the user explicitly asks the agent to submit it via `gh`.
  - Maintainer intake routes incoming reports through `ticket-triage` and `hostile-content-quarantine` (`L2` untrusted external content).
- Structure-map & meta-skill gate (`§1b`, `§1c`): consulted `.agents/skills/create-skill/SKILL.md` + `references/skill-authoring-guide.md` (`ADR 0008` Progressive Disclosure: `SKILL.md` router `<= 12` lines / `<= 220 B` description; full workflow in `references/upstream-feedback-workflow.md`; `[skill-growth-justified: ...]` recorded on commit for the new skill domain) and ran `npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc` (`exit 0`).

## The Fix

1. Author `.agents/skills/upstream-skill-feedback/SKILL.md` (lightweight router, `<= 10` lines, `description <= 220 B`) and `.agents/skills/upstream-skill-feedback/references/upstream-feedback-workflow.md` specifying:
   - Trigger: only when an installed `neo-agent-skills` / plugin skill misfires, names an unavailable tool without a standalone branch, or produces a reproducible workflow defect;
   - Redaction & consent gate: strip private repository names, file contents, secrets, and proprietary identifiers; preview the exact payload and ask once for confirmation;
   - User-owned submission (§2D): default to printing the URL-encoded pre-filled `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml` link so the user submits in their browser; execute `gh issue create --repo neomjs/neo-agent-skills` only when the user explicitly asks the agent to file it.
2. Register `upstream-skill-feedback` in `.agents/skills/skills.manifest.json` with `audiences: ["contributor", "app"]` (excluded from `maintainer` so maintainer sessions keep `ticket-create` / `defect-note` without duplicate skill-index load), wire contributor admission through dedicated `0903-contributor-skills.md`, and project into `plugins/neo` + magnet bundles (`#448`–`#451`).
3. Add `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml` (`Plugin / Skill Feedback` form: `plugin@version`, `skill`, `step`, `expected vs actual`, `harness & model`, redaction confirmation checkbox) for clean intake via `ticket-triage` + `hostile-content-quarantine`.
4. Regenerate `plugins/neo/` and `.claude-plugin/marketplace.json`, verify `npm run generate-claude-plugin -- --check`, `npm run lint`, and `npm test` (with minor version bump `2.8.0`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|
| `.agents/skills/upstream-skill-feedback/` (`SKILL.md` + `references/upstream-feedback-workflow.md`) | `ADR 0008`; Anthropic Software Directory Policy §2D & §4C | When a bundled skill misfires or lacks a standalone step, drafts a redacted report (`plugin@version`, `skill`, `step`, `expected vs actual`, `harness/model`) and after user confirmation prints a pre-filled GitHub Issue Form link for the user to submit | Runs `gh issue create` only if the user explicitly delegates filing; does nothing if user declines | `plugins/<slug>/README.md` | `npm run lint && npm test` + standalone outside-checkout smoke test |
| `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml` | `ticket-triage` & `hostile-content-quarantine` intake contract | Structured GitHub Issue Form capturing `plugin@version`, `skill`, `step`, `expected vs actual`, and `harness/model` | Standard blank issue remains available for maintainers | `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml` | YAML field-id checks + `npm test` |

Decision Record impact: aligned-with `neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract`

## Acceptance Criteria

- [ ] `.agents/skills/upstream-skill-feedback/` (`SKILL.md` `<= 12` lines / `<= 220 B` description + `references/upstream-feedback-workflow.md`) is registered in `skills.manifest.json` (`audiences: ["contributor", "app"]`), declares zero `allowed-tools` / hooks / MCP servers (`contained` reach), cites Directory Policy §2D, and defaults after redaction + consent to a pre-filled Issue Form URL (`gh issue create` only on explicit user ask).
- [ ] `.github/ISSUE_TEMPLATE/plugin-skill-feedback.yml` defines matching structured fields (`plugin@version`, `skill`, `step`, `expected vs actual`, `harness and model`) ready for `ticket-triage` and `hostile-content-quarantine` intake.
- [ ] `plugins/neo` and `.claude-plugin/marketplace.json` regenerate cleanly with `upstream-skill-feedback` bundled and pass `npm run generate-claude-plugin -- --check` and `claude plugin validate ./plugins/neo`.
- [ ] `npm run lint && npm test` exit `0` with minor-bumped `package.json` + `package-lock.json` (`2.8.0`).

## Out of Scope

- Automated telemetry, background error reporting, or filing issues without explicit per-incident user confirmation.
- Modifying `ticket-triage` or `hostile-content-quarantine` rules (both already govern external unlabeled issues).

## Avoided Traps

- **Agent-default `gh issue create` or filing without user consent / redaction:** Violates Directory Policy §2D, user trust, and `AGENTS.md §critical_gates` #9; human-submitted pre-filled Issue Form link after redaction preview is the default.
- **Adding `allowed-tools` or hooks to trigger reporting:** Downgrades plugin reach from `contained` to `privileged` in the Claude Directory.

## Related

- Parent epic: #446
- Generator prerequisites: #447 (PR #452), #453 (PR #454), #455 (PR #457)
- Downstream magnet leaves: #448, #449, #450, #451

Live latest-open + closed sweep: checked latest 20 open issues (`#387`–`#455`, `51` open total — red backlog band `> 50`, folded as required sub-issue under active epic #446) + open and closed search (`upstream OR feedback OR ISSUE_TEMPLATE OR report`, `Out of Scope` / `Avoided Traps` read on `#379`, `#400`) at `2026-11-24T23:47Z`; no equivalent or prior exclusion found.
A2A in-flight claim sweep: checked latest 30 `all`-status messages at `2026-11-24T23:47Z` (`MESSAGE:11495afa` explicitly assigns this upstream-report leaf to `@neo-gemini-julia`); zero competing claims.
MC sweep: `query_raw_memories` (`"external session skill misfire missing tool upstream issue report ISSUE_TEMPLATE"`, `5` results) + `ask_knowledge_base(type: 'ticket')`, no prior decision found.
Own-assignment sweep: `0` open assigned issues in `neomjs/neo-agent-skills`, none overlapping.

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "upstream-skill-feedback plugin-skill-feedback.yml external plugin friction gold consent contained"

## Timeline

- 2026-11-24T23:51:06Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-24T23:51:08Z @neo-gemini-julia added the `enhancement` label
- 2026-11-24T23:51:09Z @neo-gemini-julia added the `contributor-experience` label
- 2026-11-24T23:51:09Z @neo-gemini-julia added the `ai` label
- 2026-11-24T23:51:09Z @neo-gemini-julia added the `model-experience` label
- 2026-11-24T23:52:34Z @neo-gemini-julia added parent issue #446
- 2026-11-24T23:52:36Z @neo-gemini-julia marked this issue as blocking #451
- 2026-11-24T23:53:16Z @neo-gemini-julia marked this issue as blocking #448
- 2026-11-24T23:53:19Z @neo-gemini-julia marked this issue as blocking #449
- 2026-11-24T23:53:22Z @neo-gemini-julia marked this issue as blocking #450
- 2026-11-25T00:45:28Z @neo-gemini-julia cross-referenced by PR #458
- 2026-11-25T01:04:20Z @neo-gemini-julia cross-referenced by #451
- 2026-11-25T01:06:06Z @neo-gemini-julia referenced in commit `ade1dda` - "feat(skills): add upstream-skill-feedback skill and issue form (#456)

Adds the opt-in upstream-skill-feedback router and workflow, registers it across maintainer/contributor/app audiences, adds .github/ISSUE_TEMPLATE/plugin-skill-feedback.yml, and projects it into plugins/neo (v2.7.2).

[skill-growth-justified: new opt-in upstream-skill-feedback skill for external plugin and standalone sessions under #446]"
- 2026-11-25T01:06:06Z @neo-gemini-julia referenced in commit `54e67b1` - "fix(skills): scope upstream-skill-feedback to external audiences and user-actor form default (#456)

[skill-growth-justified: new opt-in upstream-skill-feedback skill for external plugin and standalone sessions under #446]"
- 2026-11-25T01:12:29Z @neo-gemini-julia cross-referenced by PR #459

