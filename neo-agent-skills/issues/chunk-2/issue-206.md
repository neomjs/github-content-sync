---
id: 206
title: 'Skill descriptions cost every session 4.6k tokens: a description budget in the corpus lint and one compaction pass'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - model-experience
assignees: []
createdAt: '2026-10-24T20:15:54Z'
updatedAt: '2026-10-24T20:15:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/206'
author: neo-fable-mnemosyne
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
---
# Skill descriptions cost every session 4.6k tokens: a description budget in the corpus lint and one compaction pass

## Context

The operator, 2026-10-24: the skills are tiny router files, so the skills line of a Claude seat's fixed boot cost (≈ 10k tokens) does not add up. Measured the same night across the 39 skills in `.agents/skills` (skills 0.1.54): the SKILL.md routers total 39,975 bytes, and **18,323 of those bytes are the frontmatter `description:` fields** (≈ 4.6k tokens), repeated in `skills.manifest.json` (18,229 bytes). The harness loads every skill's name and description into the system prompt at every session so it can decide when to invoke one; the routers themselves load only when invoked. So the routers are tiny and the cost is real: it is the descriptions, paid by every Claude seat on every boot, before any work.

## The Problem

Descriptions have grown into paragraphs. The ten largest run 584–898 bytes each (post-review-pickup 898, structural-pre-flight 876, neo-identity-update 786, epic-review 768, memory-mining 754, pull-request 632, lead-role 610, release-notes 598, ticket-create 584, ticket-intake 573); the median is ~430. They carry what the skill enforces, CRITICAL notes, empirical anchors and the trigger list — content the authoring guide places in the body: `skill-authoring-guide.md:83` asks for a "concise 1-2 sentence description of what the skill provides and when to invoke it". The corpus lint (`scripts/lint-skill-corpus.mjs`) budgets router lines (`routerByteBudget`, lines not bytes, `:503`), payload bytes (`:509`) and per-file payload bytes (`:513–522`), and checks that the frontmatter description equals the manifest row (`:490–495`); it has no budget for the description itself, so the one field every seat pays for at boot is the one field nothing bounds.

## The Architectural Reality

- `scripts/lint-skill-corpus.mjs`: defaults + per-skill overrides (`:39–40`), the budgets (`:78`), the frontmatter parse (`:490`), the manifest equality check (`:494`).
- `.agents/skills/skills.manifest.json`: 39 rows repeating each description (a shipped reference surface, `:409–425` of the lint attributes manifest text to its row).
- `.agents/skills/create-skill/references/skill-authoring-guide.md:83`: the 1–2 sentence rule; §5.3 the MCP-tool-description budget audit — the sibling discipline for tool descriptions, which already exists.
- The trade-off that must be named: the description is the harness's invocation contract. Skills #61 records that skill triggers measurably under-fire; cutting trigger clauses would make that worse. The budget therefore binds the prose, not the trigger: the trigger clause stays, the enforcement summary and anchors move into the router body.
- Design authority: `skill-authoring-guide.md:83` already states the rule; this ticket mechanises it and pays down the drift.

## The Fix

1. Lint: a `descriptionBudget` default of **220 bytes** in `lint-skill-corpus.mjs` beside the other budgets, per-skill override allowed in the manifest row like the others, applied to the frontmatter description (the manifest row is already required to be equal). An error names the skill, the bytes and the rule's line in the authoring guide.
2. Compaction pass over the 39 descriptions to one sentence of the form *what it is for + when it fires*, keeping every trigger condition as a clause and moving enforcement summaries, CRITICAL notes and anchors into the router body under a `## Triggers` or `## What it enforces` heading, where they load on invocation only. Target: ≤ 8,000 bytes total (from 18,323), ≈ 2.5k tokens saved per boot per Claude seat.
3. `skills.manifest.json` regenerated so the equality check passes; `npm run lint` and `npm test` green; the consumers' pins pick it up on the next release.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `lint-skill-corpus.mjs` budgets | the lint's own defaults block (`:39–40`, `:78`) | `descriptionBudget` (bytes) with manifest override | a row with an override above the default must name why in the manifest (`budgetRationale`), as the other overrides do | the lint's header comment | `test-lint-skill-corpus.mjs` arm: a 221-byte description fails, 220 passes, an override passes |
| the 39 `SKILL.md` frontmatter descriptions | `skill-authoring-guide.md:83` | one sentence, trigger clauses kept | — | the guide | the lint green on the compacted corpus; a before/after byte table in the PR body |
| `skills.manifest.json` | the equality check (`:494`) | regenerated | — | — | lint green |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 `npm run lint` fails on a description over `descriptionBudget` and passes at the budget; the override path is tested (`test-lint-skill-corpus.mjs`).
- [ ] AC-2 The 39 descriptions total ≤ 8,000 bytes; every trigger condition present today is still present as a clause (a diff table in the PR body lists each skill's before/after bytes and its kept triggers).
- [ ] AC-3 The manifest equality check and the materialize check pass; `npm test` green.
- [ ] AC-4 (post-merge) one Claude seat's boot shows the skills line smaller by ≥ 2k tokens against the 2026-10-24 measurement (receipt on this ticket).

## Out of Scope

Router bodies and payloads (already budgeted); the Anthropic-provided skills a harness lists on its own; the MCP tool descriptions (§5.3's audit owns those); trigger-firing reliability (#61).

## Avoided Traps

- Cutting triggers to make the number: the trade-off section above; the clause stays, the prose moves.
- A budget nobody can meet: 220 bytes fits two clauses; the current smallest (identity-firewall, 127) and the median show the shape.
- Compacting without the lint: the drift returns in a quarter.

## Related

Sibling filings tonight on the boot-cost measurement (neo-agent-brain: the handoff default, the frontier default, the mailbox projection, the boot Focus section); #61 (trigger firing); `create-skill` §5.3 (the tool-description budget, the sibling discipline).

unowned-rationale: a one-PR leaf any seat can take; the filer's week is committed (the business block); the measurement in Context is the premise. Meta-skill sweep: `create-skill` consulted; the proposal shrinks the routers' loaded surface and adds no router bloat (the moved prose loads on invocation only).

Live latest-open sweep: the latest 20 open issues, created-descending, at 2026-10-24 19:5xZ; no equivalent. A2A in-flight claim sweep: the latest 30 `[lane-claim]` rows, none on this scope. MC sweep: `skill frontmatter description length system prompt tokens every session skills list cost`, 6 results, initialization noise, no prior decision found. Own-assignment sweep: 2 open (#205, #191), none overlapping.

Origin Session ID: 542d408d-b0eb-497c-8f57-a19ca5833f5a

Retrieval Hint: "skill description budget lint 18323 bytes frontmatter boot cost tokens compaction"

## Timeline

- 2026-10-24T20:15:55Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-24T20:15:55Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-24T20:15:56Z @neo-fable-mnemosyne added the `build` label
- 2026-10-24T20:15:56Z @neo-fable-mnemosyne added the `model-experience` label

