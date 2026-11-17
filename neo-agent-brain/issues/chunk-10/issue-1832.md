---
id: 1832
title: ModelStats.md and identityRoots.mjs still record GPT-5.6 Sol for the GPT seats
state: OPEN
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-11-17T18:16:38Z'
updatedAt: '2026-11-17T18:16:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1832'
author: neo-opus-vega
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
# ModelStats.md and identityRoots.mjs still record GPT-5.6 Sol for the GPT seats

Owned and dependent on the GPT seats' runtime evidence, so it carries no curated labels.

## Context

I found this on 2026-11-17 while updating the Engine README's GPT rows to GPT-6.2 (neomjs/neo#21276), at the operator's direction after OpenAI released GPT-6.2 Astra and Sol ([announcement](https://openai.com/index/introducing-gpt-6-2-astra-sol/)). This Brain's README routes model facts away from itself: "`ai/graph/identityRoots.mjs` is the canonical identity/status registry, and `ModelStats.md` owns model facts." Both still name the July engine.

## The Problem

- `learn/agentos/ModelStats.md` `§neo_gpt` (Euclid): `name` GPT-5.6 Sol, `releaseDate` 2026-07-09, pricing $5 / $30, with sources verified 2026-07-09 to 2026-07-12. `§neo_gpt_emmy` mirrors it. There is no `@neo-gpt-sophie` section, although `identityRoots.mjs` lists her as active. The file was last updated 2026-07-29.
- `ai/graph/identityRoots.mjs:323` describes Euclid as `OpenAI Codex (GPT-5.6 Sol) Agent Identity`, and `releaseDate` / `pricingInput` / `pricingOutput` mirror `§neo_gpt`.
- In the meantime the Engine README went through GPT-6 Astra and GPT-6.1 Sol by hand (neomjs/neo#18334, neomjs/neo#20943), and through GPT-6.2 with neomjs/neo#21276. Three rotations (GPT-6, 6.1, 6.2) never reached the registry, so the rotation planner in `generateRosterOnboarding.mjs` (see #1342) finds no prior designation to match.

## The Architectural Reality

- ADR 0012 §2.3: an in-place model rotation is a rename and deprecates nothing. §2.5: every capability value cites an authoritative source, and nothing watches `sunsetTriggers` (ModelStats "Update discipline").
- Runtime fields need the bearer's own evidence: the Codex config's `model`, the catalog's context window, and the effective reasoning profile. That is how `§neo_gpt` was verified (Origin Session `f95e01ff`).

## The Fix

Once each GPT seat runs its 6.2 engine, rotate the registry in place:

- `§neo_gpt`: GPT-6.2 Sol, `releaseDate` 2026-11-17, API $2 / $0.10 cached / $10 per M tokens from the announcement, and the runtime fields from Euclid's evidence.
- `§neo_gpt_emmy`: GPT-6.2 Astra (API $8 / $0.40 / $40).
- Add `§neo_gpt_sophie`.
- Update `identityRoots.mjs`'s Euclid description and its mirrored fields, plus an `§update_history` row.

## Acceptance Criteria

- [ ] `ModelStats.md` names the engine each GPT seat runs, with a source for every value, the runtime fields cited to that seat's own evidence, and a `§neo_gpt_sophie` section.
- [ ] `identityRoots.mjs` Euclid entry: the description and the mirrored `releaseDate` / pricing match `§neo_gpt`.
- [ ] `git grep -n "GPT-5.6" -- learn/agentos/ModelStats.md ai/graph/identityRoots.mjs` returns only `§update_history` / `§sunset_history` lines.
- [ ] The identity-root and ModelStats unit specs pass.

## Out of Scope

- Making the Engine README derive from the registry. #111's drift sentinel is the standing place for that.
- The Claude, Gemini and Kimi rows.

## Related

#1342 · #111 · neomjs/neo#21276 · neomjs/neo#20943 · neomjs/neo#18334 · ADR 0012

Live latest-open + all-state sweep: latest open, plus searches for "ModelStats GPT" and "identityRoots model rotation", at 18:15Z; 156 open (red); no equivalent. #1342 (closed) fixed the planner, not the stale rows. A2A sweep: no claim. Own assignments: none here.

Origin Session ID: a1ecd0b9-0384-44fc-99f6-15bef482445f
Retrieval Hint: "ModelStats GPT-5.6 Sol stale rotation GPT-6.2 identityRoots"

## Timeline

- 2026-11-17T18:16:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-17T18:16:40Z @neo-opus-vega added the `bug` label
- 2026-11-17T18:16:40Z @neo-opus-vega added the `documentation` label
- 2026-11-17T18:16:41Z @neo-opus-vega added the `ai` label
- 2026-11-17T18:17:30Z @neo-opus-vega cross-referenced by PR #21277
- 2026-11-17T19:16:33Z @neo-opus-vega cross-referenced by #21276

