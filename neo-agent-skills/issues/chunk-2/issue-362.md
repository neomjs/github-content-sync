---
id: 362
title: The contributor guide for four repositories never links the pull-request skill
state: OPEN
labels:
  - bug
  - contributor-experience
  - ai
assignees: []
createdAt: '2026-11-11T23:48:41Z'
updatedAt: '2026-11-11T23:48:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/362'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 356
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
# The contributor guide for four repositories never links the pull-request skill

## Context

`neo-agent-skills-agents-md --repo <name> --audience contributor` prints the guide an outside author's agent reads first. Run on 2026-11-11 with 1.4.3 in an empty folder:

| `--repo` | Lines | Skills linked under "Workflows" |
|---|---|---|
| `neo` | 129 | `unit-test`, `pull-request`, `identity-firewall` |
| `neo-agent-skills`, `neo-agent-brain`, `neo-agent-institution`, `devindex` | 86 each | `identity-firewall` |

On the same day three outside authors opened pull requests in this repository (#351, #355, #357). Two of the three failed `PR body` on the `Resolves` line, which is the rule the `pull-request` skill states (`pull-request-workflow.md` §9.1) and the shared baseline enforces in every repository.

## The Problem

An outside author of four of our five repositories is told to "link it from the pull request" (§pre_commit_gates) and is never sent to the skill that says how: one standalone `Resolves #N`. Their first pull request goes red on a rule their guide did not name. The contributor projection holds `pull-request` for every repository; the guide mentions it for one.

After #360 the README's opening says the guide ends with the three skills and that another repository takes its own name after `--repo`. That sentence is true for `neo` only.

## The Architectural Reality

- Links come from the `skills:` line of the sections a guide includes (`README.md`, "Generated `AGENTS.md`").
- `agents-md/sections/0101-orientation-contributor.md` is the only contributor section with a `skills:` line for the two skills (`unit-test, pull-request`), and it declares `repos: neo`.
- Five contributor sections cover all five repositories, among them `0501-pre-commit-gates-contributor.md` and `1201-pr-diff-equals-pr-body-contributor.md`, the two that speak about pull requests. Neither declares a skill.
- `unit-test` is the Playwright workflow. Whether it applies outside the engine depends on the repository's own suite.

Design authority: "**Outside contributors can select the public author path.** `neo-agent-skills-materialize --audience contributor` (and the matching `--check`) projects `unit-test`, `pull-request` and `identity-firewall` on both surfaces." (`README.md`, "How it reaches a repository")

## The Fix

A contributor section that covers all five repositories declares `skills: pull-request`, so every guide links it. The author picks the section whose text already speaks about the pull request, and says for each of the four repositories whether `unit-test` applies there, with the command that shows it.

## Acceptance Criteria

- [ ] For each of the five repositories, `neo-agent-skills-agents-md --repo <name> --audience contributor` links `pull-request`. `scripts/test-generate-agents-md.mjs` asserts it.
- [ ] `unit-test` is linked exactly where the repository runs that suite; the pull request names the evidence per repository.
- [ ] Each generated guide stays under 23,800 bytes; `npm run lint` and `npm test` pass.
- [ ] The README's contributor line is true for every repository it names, or it is reworded in the same pull request.

## Out of Scope

- The maintainer sections.
- New contributor rules. This links a skill that is already projected.

## Avoided Traps

- **Copying the `Resolves` rule into the guide's text.** One fact, one artifact: the skill holds it, the guide links it.

## Related

#360 and its review 5866849367 (where the gap was named) · #359 (a reader who targets one of the four repositories meets it) · #310 (the contributor entry works without private tools) · #358 (the other mechanical red on the same three pull requests)

Decision Record impact: none.

unowned-rationale: Ready for a maintainer seat. It edits a generated instruction surface with a byte budget, so it is not a first issue.

Live latest-open + closed sweep: the latest 20 open by creation date (38 open in all, orange band; no ticket or epic owns the contributor sections' links) and searches over all states (`contributor guide skills link`, `agents-md contributor pull-request`; #310 and #284 read) at 2026-11-11T23:48Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the unread and the latest read messages at 23:46Z; no claim on this scope. MC sweep: "outside contributor guide for a repository other than the engine does not point to the pull-request skill, PR body check fails on the Resolves line", 5 results; Maryam's audit of 2026-11-10 on why contributors miss template and skill requirements, no ticket on this link. Own-assignment sweep: 3 open here (#356, #359, #191), not overlapping. Structure map: N/A, no file is placed.

Origin Session ID: 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

Retrieval Hint: "contributor AGENTS guide links pull-request skill only for --repo neo: sections skills declaration, Resolves rule"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session 593bf3c2-8f38-4a45-a034-cabdc74fe0f1

## Timeline

- 2026-11-11T23:48:42Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-11T23:48:43Z @neo-fable-mnemosyne added the `contributor-experience` label
- 2026-11-11T23:48:43Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T23:49:01Z @neo-fable-mnemosyne added parent issue #356
- 2026-11-11T23:49:20Z @neo-fable-mnemosyne cross-referenced by PR #360
- 2026-11-12T00:30:08Z @neo-fable-mnemosyne cross-referenced by #364
- 2026-11-12T00:44:49Z @neo-fable-mnemosyne cross-referenced by #356

