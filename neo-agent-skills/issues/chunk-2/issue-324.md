---
id: 324
title: 'lint-skill-corpus: enforce app-audience skill closure on links and prose references'
state: CLOSED
labels:
  - enhancement
  - developer-experience
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-09T14:28:10Z'
updatedAt: '2026-11-09T18:41:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/324'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-09T18:41:22Z'
---
# lint-skill-corpus: enforce app-audience skill closure on links and prose references

## Context

In `neomjs/neo-agent-skills#323` (`#312`), projecting `context-recovery` into `--audience app` (`skills.manifest.json` `"audiences": ["maintainer", "app"]`) initially passed `npm run lint` (`scripts/lint-skill-corpus.mjs`) while `.agents/skills/context-recovery/references/context-recovery-workflow.md` still contained unqualified prose references to maintainer-only skills (`memory-mining`, `session-sunset`, `pr-review`, `post-review-pickup`). `@neo-opus-grace` caught this in Round-1 review `5825517200` (`RA-1`) and noted in Round-2 review `5827186556` (`MESSAGE:63050727-be38-4f5c-bd30-dc4bf77b16a0`) that `scripts/lint-skill-corpus.mjs` currently lacks an audience-closure rule for `--audience app` skills.

Design authority: "RA-1: the app projection names only skills the app audience receives." (`neomjs/neo-agent-skills#323#pullrequestreview-5825517200` and `scripts/lint-skill-corpus.mjs:512-516` `"An app-audience skill ships into workspaces whose harness the developer chose"`).

- **Live latest-open + closed sweep:** checked latest 25 open issues (`31` open total — orange band; no overlapping open ticket) + closed search (`audience closure lint-skill-corpus app`, 0 hits) in `neomjs/neo-agent-skills` at `2026-11-09T14:27:16Z`; no equivalent or prior exclusion found.
- **A2A in-flight claim sweep:** checked `list_messages({status: "all", limit: 30})` at `2026-11-09T14:27:19Z`; `@neo-opus-grace` explicitly routed this ticket to `@neo-gemini-maryam` in `MESSAGE:63050727-be38-4f5c-bd30-dc4bf77b16a0`.
- **Memory Core rationale + own-assignment sweep:** ran `query_raw_memories({query: "lint-skill-corpus audience closure app skills manifest non-app skill references"})` and `gh issue list --repo neomjs/neo-agent-skills --state open --assignee @me` (`#312` only, in PR `#323`); no duplicate found.

## The Problem

When `scripts/materialize-harness-skills.mjs --audience app` runs in a standalone `neo.mjs` application workspace (such as one scaffolded by `npx neo-app`), it copies only the skills whose `skills.manifest.json` row includes `"app"` in `audiences` (`context-recovery`, `neo-app-authoring`, `neural-link`).

`scripts/lint-skill-corpus.mjs` currently enforces:
1. Line `514-516`: `SKILL.md` in an `app`-audience skill must not name harness-specific read tools (`view_file`, `read_file`).
2. Line `556-577`: relative Markdown links `[...](./...)` / `[...](../...)` within `.agents/skills/` must resolve on disk in `neo-agent-skills`.

However, `scripts/lint-skill-corpus.mjs` does **not** verify audience closure for `app`-audience skills:
- A relative Markdown link from an `app`-audience skill (`SKILL.md` or `references/**/*.md`) into a maintainer-only skill directory (e.g., `../../pr-review/...`) resolves in the full `neo-agent-skills` repository and passes §3, but becomes a dead link in every materialized `--audience app` workspace.
- Unqualified prose references in an `app`-audience skill (`SKILL.md` or `references/**/*.md`) to maintainer-only skill names (`memory-mining`, `session-sunset`, `pr-review`, `post-review-pickup`, etc.) instruct external app agents to invoke skills that were never materialized into their workspace unless the enclosing paragraph/bullet explicitly scopes them to `Agent OS` or `maintainer` workspaces.

## The Architectural Reality

- `scripts/lint-skill-corpus.mjs` walks every declared skill in `.agents/skills/skills.manifest.json` (`manifest.skills`).
- The set of `app`-audience skill names is `Object.keys(manifest.skills).filter(k => manifest.skills[k].audiences?.includes("app"))`, and the set of maintainer-only skill names is the complement in `Object.keys(manifest.skills)`.
- Across the current 3 `app`-audience skills (`context-recovery`, `neo-app-authoring`, `neural-link`), only `.agents/skills/context-recovery/references/context-recovery-workflow.md` mentions non-`app` skills (`memory-mining`, `session-sunset`, `pr-review`, `post-review-pickup`), and after `5bfe837` (`#323`) every one of those 5 occurrences sits inside a paragraph/bullet that explicitly qualifiers the reference with `Agent OS` / `maintainer`.
- `scripts/test-lint-skill-corpus.mjs` tests `scripts/lint-skill-corpus.mjs` rules via isolated temp-corpus fixtures.

## The Fix

1. **Cross-skill relative link closure in `scripts/lint-skill-corpus.mjs` (§3 reference integrity):**
   - When linting a file inside an `app`-audience skill (`row.audiences?.includes("app")`), if a relative Markdown link resolves inside `SKILLS` (`target.startsWith(SKILLS)`), extract the target skill name (`relative(SKILLS, target).split(sep)[0]`) and require `manifest.skills[targetSkill]?.audiences?.includes("app")`. Report a clear error if an `app`-audience skill links into a non-`app` skill directory.
2. **Prose skill-reference closure in `scripts/lint-skill-corpus.mjs`:**
   - When linting `SKILL.md` or `references/**/*.md` inside an `app`-audience skill, inspect each prose paragraph / list-item block (outside fenced code blocks) for references to non-`app` skill names (backticked `` `<skill>` ``, slash-invoked `/<skill>`, or `.agents/skills/<skill>`).
   - Require any block that names a non-`app` skill to carry an explicit `Agent OS` or `maintainer` scope qualifier; otherwise report the file, skill, and unqualified non-`app` skill name.
3. **Unit tests in `scripts/test-lint-skill-corpus.mjs`:**
   - Add regression tests covering (a) a cross-skill relative link from an `app`-audience skill to a maintainer-only skill (rejected), (b) an unqualified prose reference to a maintainer-only skill in an `app`-audience payload (rejected), and (c) a prose reference scoped with `Agent OS` / `maintainer` (accepted).

## Decision Record impact

`aligned-with` `neomjs/neo#adr:skill-anatomy-and-authoring-contract`.

## Acceptance Criteria

- [ ] `scripts/lint-skill-corpus.mjs` rejects relative Markdown links from `app`-audience skills (`SKILL.md` and `references/**/*.md`) into non-`app` skill directories.
- [ ] `scripts/lint-skill-corpus.mjs` rejects unqualified references to non-`app` skill names in `app`-audience skills while permitting references explicitly scoped to `Agent OS` / `maintainer` workspaces.
- [ ] `scripts/test-lint-skill-corpus.mjs` covers both failure modes and the qualified pass case, and `npm run lint && npm test` pass in `neomjs/neo-agent-skills`.

## Out of Scope

- Changing the `app` skill set in `skills.manifest.json` (`context-recovery`, `neo-app-authoring`, `neural-link`).
- Modifying `scripts/materialize-harness-skills.mjs`.

## Avoided Traps

- **Banning all mentions of maintainer-only skills in `app`-audience payloads:** `context-recovery-workflow.md` is shared between `maintainer` and `app` audiences and legitimately documents how Agent OS maintainer workspaces route to `memory-mining`, `session-sunset`, `pr-review`, and `post-review-pickup` alongside the standalone/app route. Requiring an `Agent OS` / `maintainer` qualifier in the enclosing block preserves single-source shared payloads while catching unscoped leaks like `#323` `RA-1`.

## Related

- Follow-up to `neomjs/neo-agent-skills#312` / PR `neomjs/neo-agent-skills#323` (`RA-1` from `@neo-opus-grace` in review `5825517200` and `5827186556`).

Origin Session ID: 44dae607-d2f7-497c-951b-5b93c0d94ab0
Retrieval Hint: "lint-skill-corpus audience closure app skills non-app skill references RA-1 #312 #323"

## Timeline

- 2026-11-09T14:28:12Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-09T14:28:12Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-09T14:28:12Z @neo-gemini-maryam added the `developer-experience` label
- 2026-11-09T14:28:13Z @neo-gemini-maryam added the `ai` label
- 2026-11-09T14:28:13Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-09T14:28:13Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-09T14:32:54Z @neo-gemini-maryam cross-referenced by PR #325
- 2026-11-09T14:45:11Z @neo-gemini-maryam referenced in commit `f4fe663` - "test(lint): pin each detection form and narrow qualifier to Agent OS (#324)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-09T15:52:46Z @neo-gemini-maryam referenced in commit `9156171` - "feat(lint): enforce app-audience skill closure on links and prose references (#324)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-09T15:52:47Z @neo-gemini-maryam referenced in commit `acf6c2d` - "test(lint): pin each detection form and narrow qualifier to Agent OS (#324)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-09T18:41:23Z @tobiu referenced in commit `feeabeb` - "feat(lint): enforce app-audience skill closure on links and prose references (#324) (#325)

* feat(lint): enforce app-audience skill closure on links and prose references (#324)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>

* test(lint): pin each detection form and narrow qualifier to Agent OS (#324)

Co-Authored-By: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-09T18:41:23Z @tobiu closed this issue

