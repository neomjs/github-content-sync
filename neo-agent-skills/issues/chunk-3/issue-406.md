---
id: 406
title: Align pull-request workflow §5/§9 and add canonical PR template
state: CLOSED
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-15T23:23:10Z'
updatedAt: '2026-11-16T00:35:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/406'
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
closedAt: '2026-11-16T00:35:34Z'
---
# Align pull-request workflow §5/§9 and add canonical PR template

## Context

Phase-2 follow-up to `neomjs/neo#21055` / `neomjs/neo#21056` (satisfying `RA-4` from cross-family review `5913424583`), gated on cross-family peer approval of `neomjs/neo#21056` per operator `2026-11-15` rollout sequence ("start with engine, wait until a peer approves, then open the other ones").

While Engine aligns `.github/PULL_REQUEST_TEMPLATE.md` and `CONTRIBUTING.md` in Phase 1, the shared Skills repository (`neomjs/neo-agent-skills`) still carries two contributor-template lines in `.agents/skills/pull-request/references/pull-request-workflow.md` that become stale on merge, and lacks `.github/PULL_REQUEST_TEMPLATE.md` itself. Sibling repositories `neomjs/neo-agent-brain` and `neomjs/neo-agent-institution` also currently lack `.github/PULL_REQUEST_TEMPLATE.md` (both already have `CONTRIBUTING.md`) and follow after `neomjs/neo#21056` is approved.

## The Problem

- **Stale `pull-request-workflow.md` §5 contributor disclosure line (line 162):** Currently states only `**Contributor:** use the actual public author and disclose AI model/harness assistance.` without specifying the canonical comma-separated `Authored by @<github-handle> (<model + version>, reasoning: <effort or default>, <agent or harness>).` (or `(no LLM)`) format and its cross-rival-lab review routing rationale.
- **Stale `pull-request-workflow.md` §9 template sentence (line 299):** Currently states `Do not copy ticket bodies or the optional external-contributor \`.github/PULL_REQUEST_TEMPLATE.md\` into agent PRs; summarize the implementation delta below.` — once `.github/PULL_REQUEST_TEMPLATE.md` IS the canonical `§9.1` skeleton, describing it as a divergent external-only template contradicts `pull-request/SKILL.md` ("Both audiences use the canonical body and factual attribution").
- **Missing `.github/PULL_REQUEST_TEMPLATE.md` in `neomjs/neo-agent-skills`:** Contributors opening PRs in this repo's GitHub web UI receive no pre-filled `§9.1` skeleton or cross-rival-lab model/version/reasoning disclosure prompt.
- **Missing Skills package bump:** `.agents/skills/**` changes in `neomjs/neo-agent-skills` require bumping `version` (`package.json` + `package-lock.json`) per `check-version-bump.mjs` before downstream consumers inherit the updated guidance.

## The Architectural Reality

Design authority: `"Both audiences use the canonical body and factual attribution; private identity, routing and lifecycle requirements apply to maintainers only."` (`.agents/skills/pull-request/SKILL.md`), together with operator `2026-11-15` cross-rival-lab disclosure ruling (`Authored by ` specifies model, exact version, reasoning effort / `default` fallback, and agent/harness).

Structure-map gate: `npm run ai:structure-map -- --files --loc` ran in `neomjs/neo-agent-brain` (no Brain `ai/` module moves); the Skills skill payload remains in `.agents/skills/pull-request/references/pull-request-workflow.md` (`SKILL.md` router stays 10 lines under ADR `0008`), and `.github/PULL_REQUEST_TEMPLATE.md` lives at repository root matching `neomjs/neo`.

- `.agents/skills/pull-request/references/pull-request-workflow.md:158-169` (`## 5. Self-Identification`) and `:297-302` (`## 9. PR Body Hygiene`) are read by both contributors and maintainers (note `#362` recently wired `--audience contributor` guides in all repos to link `pull-request`).
- Sibling `neomjs/neo-agent-skills#400` / `#398` own contributor-assistance wording in `README.md` (explicitly noting this repo had no root `.github/PULL_REQUEST_TEMPLATE.md` yet); adding the template preserves `README.md` untouched and avoids collision.
- Sibling alignment PRs in `neomjs/neo-agent-brain` and `neomjs/neo-agent-institution` (adding `.github/PULL_REQUEST_TEMPLATE.md` and aligning `CONTRIBUTING.md`) live in their own repositories once `neomjs/neo#21056` is approved and are tracked in `## Post-Merge Validation` below.

## The Fix

### Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/pull-request/references/pull-request-workflow.md` §5 (line 162) | `.agents/skills/pull-request/SKILL.md`, operator `2026-11-15` ruling, `neomjs/neo#21055` | Updates `**Contributor:**` to ask for `Authored by @<github-handle> (<model + version>, reasoning: <effort or default>, <agent or harness>).` (comma-separated, no vendor keywords in template comments, no `/` between dotted version and effort) and states cross-rival-lab review routing rationale. | `(no LLM)` when no LLM assisted; `reasoning: default` when the tool exposes no effort setting. | Self-documenting in `pull-request-workflow.md` §5 | Diff + `npm test` + `npx neo-agent-skills-pr-body` |
| `.agents/skills/pull-request/references/pull-request-workflow.md` §9 (line 299) | `neomjs/neo#21055` / `neomjs/neo#21056`, review `5913424583` `RA-4` | Retires `"the optional external-contributor .github/PULL_REQUEST_TEMPLATE.md"` exception while keeping the rule never to copy ticket bodies and forbidding unfilled `<...>` placeholders (``Do not copy ticket bodies or leave `.github/PULL_REQUEST_TEMPLATE.md` `<...>` placeholders unfilled; summarize what shipped below.``). | Keep all existing `§9.1` required anchors intact. | `pull-request-workflow.md` §9 | Diff + `npm test` + `npx neo-agent-skills-guides` |
| `.github/PULL_REQUEST_TEMPLATE.md` (new) | `neomjs/neo#21056` approved Engine template | Pre-fills the unified `§9.1` skeleton with placeholder null states, linked Evidence Ladder, `reasoning: <effort or default>`, and cross-rival-lab disclosure HTML comment containing zero vendor/model words. | Contributors without an LLM use `(no LLM)`. | HTML comments + `CONTRIBUTING.md` (other repos) | `disclosedFamiliesOf()` returns `[]` raw, exact family filled; `npx neo-agent-skills-pr-body` exits `0` |
| `package.json`, `package-lock.json` | `scripts/check-version-bump.mjs` | Patch-bump `neo-agent-skills` package version (`1.9.11` -> `1.9.x`). | None; CI fails without bump. | `package.json` | `npm test` + CI `check-version-bump` pass |

Decision Record impact: none

## Acceptance Criteria

- [ ] AC-1: `.agents/skills/pull-request/references/pull-request-workflow.md` §5 aligns `**Contributor:**` with comma-separated `Authored by @<github-handle> (<model + version>, reasoning: <effort or default>, <agent or harness>).` (or `(no LLM)`) and states the cross-rival-lab review routing rationale.
- [ ] AC-2: `.agents/skills/pull-request/references/pull-request-workflow.md` §9 replaces the stale `"optional external-contributor .github/PULL_REQUEST_TEMPLATE.md"` clause while preserving the rule not to copy ticket bodies.
- [ ] AC-3: `.github/PULL_REQUEST_TEMPLATE.md` is added matching the cross-family-approved `neomjs/neo#21056` Engine template, passes `npx neo-agent-skills-pr-body --body-file` once a single `Resolves #<N>` number is supplied, and `disclosedFamiliesOf({body})` returns `[]` on the raw unfilled template.
- [ ] AC-4: `package.json` and `package-lock.json` bump `version`, and `npm test` plus `npm run lint` pass (verified before opening, after `neomjs/neo#21056` cross-family approval).

## Post-Merge Validation

- [x] Verify Phase-2 sibling alignment PRs in `neomjs/neo-agent-brain` ([`#1775`](https://github.com/neomjs/neo-agent-brain/issues/1775) / [`#1776`](https://github.com/neomjs/neo-agent-brain/pull/1776)) and `neomjs/neo-agent-institution` ([`#993`](https://github.com/neomjs/neo-agent-institution/issues/993) / [`#994`](https://github.com/neomjs/neo-agent-institution/pull/994)) are opened and cross-linked once `neomjs/neo#21056` receives cross-family approval.

## Out of Scope

- Opening this Skills PR or the Brain / Institution Phase-2 PRs before `neomjs/neo#21056` receives cross-family peer approval (per `2026-11-15` operator staging rule).
- Editing `SKILL.md` router frontmatter/body (respects ADR `0008` router byte budget; updates live in `references/pull-request-workflow.md`).
- Editing `README.md` or duplicating `neomjs/neo-agent-skills#400`'s contributor-assistance entry work.

## Avoided Traps

- **Omitting `good first issue` / `help wanted`:** Intentionally omitted because this is an operator-sequenced Phase-2 follow-up already owned by `@neo-gemini-maryam` after `neomjs/neo#21056` (`ticket-create-workflow.md §4`).
- **Premature implementation before Engine approval:** Filing this ticket satisfies review `5913424583` `RA-4` ownership cleanly without breaking the operator's `Engine -> Brain/Skills/FM` PR staging gate.
- **Naming vendor or model examples inside `.github/PULL_REQUEST_TEMPLATE.md` or `pull-request-workflow.md` template snippets:** Keep placeholders neutral (`<model + version>`) so comment-scanning parsers (`disclosedModelFamilies.mjs`) never false-positive.

## Related

- `neomjs/neo#21055` / `neomjs/neo#21056` — Merged Phase-1 Engine PR template & `CONTRIBUTING.md` alignment (`5913524647`, `5913424583` `RA-4`)
- `neomjs/neo-agent-brain#1775` / `neomjs/neo-agent-brain#1776` — Phase-2 Brain `.github/PULL_REQUEST_TEMPLATE.md` & `CONTRIBUTING.md` rollout
- `neomjs/neo-agent-institution#993` / `neomjs/neo-agent-institution#994` — Phase-2 Institution `.github/PULL_REQUEST_TEMPLATE.md` & `CONTRIBUTING.md` rollout
- `#362` — Closed predecessor noting in comment that `pull-request-workflow.md` §5's contributor disclosure needed a fixed format follow-up
- `#400` / `#398` — Orthogonal contributor-assistance guidance in Skills `README.md`
- `#14` — Umbrella Epic for cross-repo PR governance

Origin Session ID: 76097b8c-6818-4c1f-a0c1-abf162c6100c

Retrieval Hint: "Align pull-request-workflow §5 §9 PULL_REQUEST_TEMPLATE Phase 2 cross-rival-lab disclosure"

Live latest-open + closed sweep: checked latest 20 open by creation + open and closed search (`pull-request-workflow OR PULL_REQUEST_TEMPLATE`, read `#362` and `#399` `Out of Scope` / `Avoided Traps` plus open `#400`/`#401`/`#398`/`#14`/`#405`) at 2026-11-15T23:21Z; 38 open (orange band); no equivalent ticket found (closed `#362` comment explicitly noted this follow-up was not yet filed).
A2A in-flight claim sweep: scanned 30 most recent messages across all read states via `list_messages` at 2026-11-15T23:21Z; no concurrent claim on `pull-request-workflow` §5/§9 or `.github/PULL_REQUEST_TEMPLATE.md`.
MC sweep: `query_raw_memories` (`"PULL_REQUEST_TEMPLATE external contributor hybrid PR body pull-request-workflow disclosedModelFamilies"`), 5 results, no prior exclusion found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-skills` (`gh issue list --assignee @me`).


## Timeline

- 2026-11-15T23:23:10Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-15T23:23:11Z @neo-gemini-maryam added the `documentation` label
- 2026-11-15T23:23:11Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-15T23:23:11Z @neo-gemini-maryam added the `contributor-experience` label
- 2026-11-15T23:23:11Z @neo-gemini-maryam added the `ai` label
- 2026-11-15T23:34:57Z @neo-gemini-maryam cross-referenced by PR #21056
- 2026-11-15T23:58:22Z @neo-gemini-maryam cross-referenced by PR #407
- 2026-11-16T00:35:34Z @tobiu referenced in commit `3bcf88b` - "docs(pull-request): align §5/§9 with canonical PR template (#406) (#407)"
- 2026-11-16T00:35:36Z @tobiu closed this issue

