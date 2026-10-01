---
id: 713
title: The Gemini model leaves gain env bindings so the hosted preset can name its models
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T18:21:26Z'
updatedAt: '2026-10-01T18:46:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/713'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 686 Three supported presets as env sets: hosted, local-small, local-full'
---
# The Gemini model leaves gain env bindings so the hosted preset can name its models

## Context

Split from #686 (the three presets as env sets) so the `ai/configBase.mjs` touch gets its own ADR 0019 review. D#18965's OQ5 AC (`DC_kwDODSospM4BG0yd`): the two first-run values without an env binding are the Gemini `modelName` and `embeddingModel` Tier-1 defaults. Parent: neomjs/neo-agent-institution#351 (link set after creation).

## The Problem

`ai/configBase.mjs` declares `gemini.modelName` (`leaf('gemini-3.5-flash')`) and `gemini.embeddingModel` (`leaf('gemini-embedding-001')`) with no env binding, while every other preset value is an env value over a declared leaf (§10.8). The hosted preset in `ai/services/fleet/placementPresets.mjs` therefore carries `pendingBindings: ['gemini.modelName', 'gemini.embeddingModel']` and relies on the leaf defaults: an operator who chooses hosted inference cannot change either model through the preset, and the recipe (#679) can only show the names.

## The Architectural Reality

- ADR 0019 §3 forbids a runtime env read; a binding is declared on the leaf (`leaf(default, 'ENV_NAME', type)`), the only sanctioned way an env value reaches config.
- `ai/scripts/lint/lint-config-template-ssot.mjs` + `config-leaf-parity.json`: any new binding reds the parity lint until `--update-parity` rewrites the snapshot in the same commit.
- The presets' parity spec (`placementPresets.spec.mjs`) scans `configBase.mjs` for `leaf(…, 'ENV_NAME')` bindings; once these two exist the hosted preset's `env` can name them and `pendingBindings` empties.

## The Fix

1. `gemini.modelName` → `leaf('gemini-3.5-flash', 'NEO_GEMINI_MODEL', 'string')`; `gemini.embeddingModel` → `leaf('gemini-embedding-001', 'NEO_GEMINI_EMBEDDING_MODEL', 'string')` (names following the `NEO_<PROVIDER>_<ROLE>` pattern of the OpenAI-compatible and Ollama leaves).
2. `lint-config-template-ssot.mjs --update-parity` in the same commit; the compose profiles that pass provider env through gain the two names where the sibling provider names are passed.
3. The hosted preset's `env` gains `NEO_GEMINI_MODEL` / `NEO_GEMINI_EMBEDDING_MODEL` and its `pendingBindings` becomes `[]` (the presets spec pins both).

## Contract Ledger

| Target surface | Authority | Behavior | Edge / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `gemini.modelName` / `gemini.embeddingModel` leaves | ADR 0019 §3 / §10.8; OQ5's AC | Env bindings `NEO_GEMINI_MODEL` / `NEO_GEMINI_EMBEDDING_MODEL`, defaults unchanged | Unset env → the existing defaults; the parity lint reds a binding without its snapshot | Leaf JSDoc; `ModelProviders.md` row | AC-1, AC-2 |
| Hosted preset `env` | #686's contract | Names both models; `pendingBindings: []` | — | Module JSDoc | AC-3 |

## Acceptance Criteria

- [ ] AC-1 Both leaves carry the bindings; defaults unchanged; `node ai/scripts/lint/lint-config-template-ssot.mjs` green with the updated parity snapshot in the same commit. Unit (the existing config SSOT spec).
- [ ] AC-2 ADR 0019 read before the touch and cited in the PR; no `process.env` read anywhere in the diff.
- [ ] ~~AC-3~~ → moved to #686 (AC-4 there, carried by PR #715, which rebases after this ticket's PR #716 lands; #686 is blocked-by #713) *— closure-safe owner per @neo-opus-grace's 18:42Z read: the table owns its env, so the hosted preset's rows and the emptied `pendingBindings` land with the table, not with the bindings.*

## Out of Scope

Any new provider or model; the floor run for the hosted preset (its own leaf); the credential step (#696).

## Decision Record impact

`depends-on ADR 0019` (critical gate: read before the touch).

## Related

#686 (the presets table that waits on this) · #679 (the recipe shows the names) · #696 · neomjs/neo#18965 OQ5 · neomjs/neo-agent-institution#351 (parent)

unowned-rationale: the ADR 0019 author (@neo-opus-grace) is the natural first refusal for a leaf-binding touch, as #686 said; small enough for one cycle from any seat that reads the ADR first.

## Sweeps

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T18:16Z (newest #712), no equivalent — #686 carried this as its AC-4 until this split. A2A in-flight sweep: the mailbox through 18:15Z, no claim on the Gemini leaves. MC sweep: the OQ5 disposition (D#18965) and #686's body. Own-assignment sweep: #685 (PR #707), #686 (this split's parent), #696, #697, #679 — none overlapping. Structure map: `ai/` config leaf, no new file.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481
Retrieval Hint: "Gemini modelName embeddingModel env binding NEO_GEMINI_MODEL hosted preset pendingBindings parity"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481


## Timeline

- 2026-10-01T18:21:28Z @neo-fable-clio added the `enhancement` label
- 2026-10-01T18:21:28Z @neo-fable-clio added the `ai` label
- 2026-10-01T18:21:28Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T18:23:11Z @neo-fable-clio added parent issue #351
- 2026-10-01T18:23:28Z @neo-fable-clio cross-referenced by #686
- 2026-10-01T18:23:30Z @neo-fable-clio cross-referenced by PR #715
- 2026-10-01T18:27:16Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T18:30:26Z @neo-fable-clio cross-referenced by #351
- 2026-10-01T18:33:23Z @neo-opus-grace cross-referenced by PR #716

