---
id: 433
title: Document local neo-agent-skills-pr-body preflight and blob?raw=true embeds in pull-request-workflow.md
state: OPEN
labels:
  - documentation
  - enhancement
  - good first issue
  - help wanted
  - ai
  - model-experience
assignees: []
createdAt: '2026-11-21T10:54:03Z'
updatedAt: '2026-11-21T11:43:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/433'
author: neo-gemini-julia
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# Document local neo-agent-skills-pr-body preflight and blob?raw=true embeds in pull-request-workflow.md

Please comment to claim before starting; check current assignees and linked PRs.

## Context

While opening `neomjs/neo#21774` (`Resolves neomjs/neo#21502`), `reusable-pr-baseline.yml` (`scripts/check-pr-body.mjs` / `neo-agent-skills-pr-body`) failed on `PR Baseline` (`run 44894962816`) with:

```text
❌ Agent PR body is missing required template anchors.
   First missing: image embed `https://raw.githubusercontent.com/neomjs/media/c8ea4f8866d5371891cf3782c35f0e61ef5e9f23/neo/21502-input-masks/before-1440x900-light.png` 404s on private repos — use `https://github.com/neomjs/media/blob/c8ea4f8866d5371891cf3782c35f0e61ef5e9f23/neo/21502-input-masks/before-1440x900-light.png?raw=true`
   See .agents/skills/pull-request/references/pull-request-workflow.md §9.
```

Editing the PR body re-queued `PR Baseline`, `File Size Guard`, and `Claim responder` (3 runner jobs under the 20-slot org ceiling tracked in `D#21725`) and triggered a Fleet `[open-work:repair-ci]` wake cycle.

## The Problem

Two author-facing gaps in `.agents/skills/pull-request/references/pull-request-workflow.md` cause avoidable `PR Baseline` failures and runner churn across consumer repositories (`neomjs/neo`, `neomjs/neo-agent-institution`, `neomjs/neo-stage`, `neomjs/devindex`, `neomjs/neo-agent-skills`):

1. **Undocumented `blob/<ref>/<path>?raw=true` image-embed rule in §9:** Issue `#271` added `findImageEmbedViolations()` to `scripts/check-pr-body.mjs` (`L392-L409`), whose CLI failure footer (`L580`) directs authors to `.agents/skills/pull-request/references/pull-request-workflow.md §9`. However, `#271` touched zero `.agents/skills/**` files: the `raw.githubusercontent.com` prohibition appears only in the reviewer-side `.agents/skills/pr-review/audits/rendered-surface-audit.md:14`, never in the author-side `pull-request-workflow.md §9` (`## Test Evidence`, `L347-L351`).
2. **Missing local `npx neo-agent-skills-pr-body --body-file <draft-body.md>` pre-open check in §1:** `pull-request-workflow.md` §1 (`L24-L37`) instructs authors to run Brain-hosted `npm run agent-preflight`, then states *"a consumer without `ai/` has no such script"* (`L29`), leaving non-Brain PR bodies completely unvalidated before `gh pr create` even though every consumer installs `neo-agent-skills` and already has `./node_modules/.bin/neo-agent-skills-pr-body` (`scripts/check-pr-body.mjs`) available offline (`<50 ms`).

## The Architectural Reality

- `scripts/check-pr-body.mjs` (`bin`: `neo-agent-skills-pr-body` in `package.json`) is the pure offline PR-body validator executed by `.github/workflows/reusable-pr-baseline.yml`.
- `.agents/skills/pull-request/references/pull-request-workflow.md` (`28,107` bytes, governed by `scripts/lint-skill-corpus.mjs` `COMBINED_BUDGETS` and `AGENTS.md §self_evolving_systems` Accretion Defense) is the canonical author-side payload read before `gh pr create`.
- Structure map (`npm run ai:structure-map -- --files --loc` in `neomjs/neo-agent-brain`): `N/A` in `neo-agent-skills` runtime source; author-side PR discipline lives in `.agents/skills/pull-request/references/pull-request-workflow.md`.
- Design authority: `"scripts/check-pr-body.mjs:392 @summary Refuses raw.githubusercontent.com image embeds, which 404 for signed-in viewers on private repos."` and `"#271 The Architectural Reality: local runs: npm exec --package=neo-agent-skills@<version> -- neo-agent-skills-pr-body --body-file <draft>."`

## The Fix

Update `.agents/skills/pull-request/references/pull-request-workflow.md` while net-reducing or holding byte-neutral its `28,107`-byte footprint (`Accretion Defense`, tightening the 9-line `agent-preflight` fallback prose at `L29-L37`):

1. **In §1 (`L24-L37`):** When `npm run agent-preflight` is absent (any consumer without `ai/`), instruct authors to run `npx neo-agent-skills-pr-body --body-file <draft-body.md>` (exit `0` required before `gh pr create`) while keeping the change-class declaration line.
2. **In §9 (`## Test Evidence`, `L347-L351`):** Document in one concise clause that GitHub-hosted image embeds must use `https://github.com/<owner>/<repo>/blob/<ref>/<path>?raw=true` (never `raw.githubusercontent.com`, which `neo-agent-skills-pr-body` refuses because it `404`s on private repositories).
3. Run `npm test` and `npm run lint` (`scripts/lint-skill-corpus.mjs`) to verify router/payload byte budgets pass.

Decision Record impact: `aligned-with neomjs/neo#adr:0007-agents-md-compaction-taxonomy` and `aligned-with neomjs/neo#adr:0008-skill-anatomy-and-authoring-contract` (keeps `SKILL.md` router untouched and trims §1 prose so `pull-request-workflow.md` does not accrete bytes).

## Acceptance Criteria

- [ ] AC-1: `.agents/skills/pull-request/references/pull-request-workflow.md` §1 directs non-Brain consumers (where `npm run agent-preflight` is absent) to run `npx neo-agent-skills-pr-body --body-file <draft-body.md>` before `gh pr create`.
- [ ] AC-2: `.agents/skills/pull-request/references/pull-request-workflow.md` §9 (`## Test Evidence`) states the `https://github.com/<owner>/<repo>/blob/<ref>/<path>?raw=true` image-embed rule (forbidding `raw.githubusercontent.com`) that `scripts/check-pr-body.mjs` enforces and cites.
- [ ] AC-3: `.agents/skills/pull-request/SKILL.md` router is unchanged, `pull-request-workflow.md` byte size is `<= 28,107` bytes (`Accretion Defense`), and `npm test` + `npm run lint` pass.

## Out of Scope

- Changing `scripts/check-pr-body.mjs` validation rules (already shipped in `#271`).
- Modifying `.agents/skills/pull-request/SKILL.md` router lines or packaging Brain-only `collectStaleOverlayFindings` into `neo-agent-skills`.

## Avoided Traps

- **Adding prose to `SKILL.md` or growing `pull-request-workflow.md` past its combined byte budget:** Trim the verbose 9-line `agent-preflight` fallback paragraph (`L29-L37`) when adding the `npx neo-agent-skills-pr-body` command and `blob/...?raw=true` clause so total loaded bytes stay `<= 28,107`.

## Related

- `#271` — Shipped `findImageEmbedViolations` in `scripts/check-pr-body.mjs` without updating `pull-request-workflow.md` §9
- `#14` — Umbrella Epic: Unify PR governance across Neo repositories
- `#406` — Prior §5/§9 alignment in `pull-request-workflow.md`
- `neomjs/neo#21774` — Empirical `PR Baseline` failure (`44894962816`) + 3-job rerun specimen under `D#21725`

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked latest 30 open (`40` open — orange band, linked to `#14` / `#271`) + open and closed search (`raw.githubusercontent OR image embed OR pull-request-workflow`, `neo-agent-skills-pr-body`, read `Out of Scope` and `Avoided Traps` on `#271` and `#406`, plus `#14`) at `2026-11-21T10:52Z`; no equivalent open ticket or exclusion found (`#271` explicitly left `.agents/skills` untouched).
A2A in-flight claim sweep: latest 30 messages across all read-states at `2026-11-21T10:52Z`; no overlapping claim.
MC sweep: `"agent-preflight not hosted here PR Baseline check-pr-body raw.githubusercontent rerun"`, 5 results, no prior exclusion found.
Own-assignment sweep: 2 open (`#432`, `#423`), bodies checked, none overlapping.

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181

Retrieval Hint: "pull-request-workflow §1 §9 neo-agent-skills-pr-body raw.githubusercontent blob raw=true pre-open"

## Timeline

- 2026-11-21T10:54:04Z @neo-gemini-julia added the `documentation` label
- 2026-11-21T10:54:04Z @neo-gemini-julia added the `enhancement` label
- 2026-11-21T10:54:04Z @neo-gemini-julia added the `good first issue` label
- 2026-11-21T10:54:04Z @neo-gemini-julia added the `help wanted` label
- 2026-11-21T10:54:05Z @neo-gemini-julia added the `ai` label
- 2026-11-21T10:54:05Z @neo-gemini-julia added the `model-experience` label
### @nazarijlarin402-pixel - 2026-11-21T11:43:18Z

Hi! I can help with your posted task: Document local neo-agent-skills-pr-body preflight and blob?raw=true embeds in pull-request-workflow.md. I can deliver Draft software or process documentation after we agree on the exact input and output. I use AI-assisted tools, with review and a concrete ZIP/file deliverable where appropriate. Could you confirm your proposed budget and the output you need? We can confirm scope, AI-use policy, and payment terms before work begins. Work will not begin until we agree.

Original GIA examples (demos, not past client work): [QUARANTINED_URL: github.com]


