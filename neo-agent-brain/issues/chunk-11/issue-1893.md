---
id: 1893
title: The Brain lints neither its guides nor Engine refs to its scripts
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-22T03:18:12Z'
updatedAt: '2026-11-22T05:35:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1893'
author: neo-opus-grace
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
closedAt: '2026-11-22T05:35:29Z'
---
# The Brain lints neither its guides nor Engine refs to its scripts

## Context

Split from #1518 by its intake re-measure (2026-11-22, Brain `dev` `d6f43108`). #1518 keeps the doubled check names and the ruleset edit they need. This ticket takes the guide-lint half, which renames no required context.

## The Problem

Measured at Brain `dev` `d6f43108` and neo `dev` `c916b512ee`:

- **The Brain's guides get no guide lint.** `shared-pr-baseline.yml` passes no `guides:` input, and no Brain workflow or package script runs one (`git grep -E 'guides|lint-guides' -- .github/`, with `Shared PR Baseline` as the control hit, plus a read of `package.json`).
- **neo's baseline counts on a Brain run that does not exist.** neo `.github/workflows/pr-baseline.yml:34-35` skips `dead-script-ref` because its guides "name the Brain's `ai:*` scripts", and leaves that rule to the Brain at its `neo.mjs` pin. Nothing in the Brain runs it. The Engine's `learn/agentos/RunningTheFleetCockpit.md` names `npm run ai:fleet-server` four times, both at the pin and on neo `dev`. Neither of the Engine's manifests at the pin defines that script, and neither does the Brain's `package.json`: only the Brain's `cloud/package.json` does. A Brain PR that renames it merges green in both repositories, and the Engine guide goes on naming a dead command.
- **A copy of the lint runs nowhere.** `ai/scripts/lint/lint-guides.mjs` (651 lines) and `lintGuides.spec.mjs` (338) duplicate `neo-agent-skills-guides`. The bin's port also removed seven quadratic paths the copy still has (neomjs/neo-agent-skills#321, `e54f994`). Besides its own spec, only `isEntryModule.spec.mjs` uses it. No tracked code, workflow or package script in nine other organization repositories names it (neo, neo-agent-skills, neo-agent-institution, devindex, pages, middleware-v2, create-app, github-content-sync, neo-stage; each default branch, 2026-11-22; the Brain's three hits are the control).

## The Architectural Reality

Design authority: "The Brain runs that rule over these guides at its own `neo.mjs` pin." (neo `.github/workflows/pr-baseline.yml:35`)

- `reusable-pr-baseline.yml@v2.3.0` runs a `Guides` step inside `PR baseline` when a caller passes `guides: true`, and only over the checkout root. `Shared PR Baseline / PR baseline` is a required context on Brain `dev` (`gh api repos/neomjs/neo-agent-brain/rules/branches/dev`, 2026-11-22), so a HARD finding blocks the merge.
- The Engine-at-pin run therefore needs a step of the Brain's own. `neo-agent-skills-guides` takes `--root`, a repeatable `--script-root` (each root's `package.json` and `cloud/package.json`), `--openapi-root` and `--skip`. The Brain depends on `neo-agent-skills@^2.3.0` and pins `neo.mjs` to an Engine commit.
- `brain-lint.yml` is the Brain's lint home: one job with one step per lint, path-filtered on what its lints scan, and deliberately not a required context. Its paths already include `package.json` and `cloud/package.json`. `lintWorkflowScanRootParity.spec.mjs` registers each lint's scan surface.
- Measured with the 2.3.0 bin from the Brain root:
  - **The Brain's own guides:** 26 full guides, 0 hard, 22 warnings, exit 0. The same holds at #1892's head `91a6115c8a`, and the copy reports the same numbers.
  - **The Engine's guides at the pin**, with `--root node_modules/neo.mjs --script-root . --script-root node_modules/neo.mjs --openapi-root .` and the eight non-data rules skipped: 10 full guides, 0 hard, exit 0. The same run without `--script-root .` reports 4 HARD `dead-script-ref` findings on `ai:fleet-server` and exits 1, so the step catches the rename.
- `isEntryModule.spec.mjs:84` needs a Brain CLI that prints when it is launched through a symlinked checkout path. `lint-identity-vocabulary.mjs` fits: it is install-free, gates its CLI block on `isEntryModule`, and prints its one-line report when launched that way.

## The Fix

1. `shared-pr-baseline.yml`: add `guides: true` under `with:`.
2. `package.json`: an `ai:lint-engine-guides` script runs the bin over `node_modules/neo.mjs`. The Brain and the pin are its script roots and the Brain is its OpenAPI root. `--skip` names every rule except `dead-script-ref` and `openapi-tool-parity`. `brain-lint.yml` runs it in a step after `Install dependencies`. The parity spec registers it with `package.json` as its script, because the manifest defines the invocation and pins the Engine. Its declared surface is `package.json`, `package-lock.json` (which resolves the bin's version and the pin), `cloud/package.json` and `ai/mcp/server/*/openapi.yaml`. `package-lock.json` joins both of the workflow's path lists.
3. Delete `ai/scripts/lint/lint-guides.mjs` and `test/playwright/unit/ai/scripts/lint/lintGuides.spec.mjs`. The deleting commit says where the assertions live now (`scripts/test-check-guides.mjs` in neo-agent-skills), in the Engine's `spec-retired:` form, since the Brain runs no such guard.
4. `isEntryModule.spec.mjs`: launch `lint-identity-vocabulary.mjs` instead.

#1892 edits `brain-lint.yml` and the parity spec too, in other hunks, and its head lints green under `guides: true`, so either can merge first.

## Acceptance Criteria

- [ ] AC-1: `Shared PR Baseline / PR baseline` runs the `Guides` step over the Brain's guides with 0 hard at the PR head, and a HARD finding fails that context (red-first evidence in the PR).
- [ ] AC-2: The new `brain-lint.yml` step fails when the Brain stops defining an `ai:*` script that an Engine guide names at the pin (red-first: the same run without the Brain as a script root).
- [ ] AC-3: `lint-guides.mjs` and its spec are gone, and no tracked Brain file names either.
- [ ] AC-4: `isEntryModule.spec.mjs` passes against `lint-identity-vocabulary.mjs`, and fails when `isEntryModule` stops resolving symlinks (red-first).
- [ ] AC-5: `lintWorkflowScanRootParity.spec.mjs` passes with the new step registered.

## Out of Scope

- The doubled check names and the ruleset edit (#1518).
- What any rule checks, and the Engine's lint of its own guides.
- neo-agent-skills `guide-authoring-bar.md:54` credits a HARD failure to `ai:lint-guides`, a script no repository defines. That line is a skills-repository fix, captured as a defect-note.

## Related

Split from #1518, whose AC-3 and AC-4 move here. neomjs/neo#20597 (the Engine side of the guide lint) · neomjs/neo-agent-skills#321 (the bin) · #1360 and #1366 (the copy's root options, which the bin carries) · #1368 (`isEntryModule`) · #1891 and its PR #1892 (the same two files).

Decision Record impact: none (aligned with neomjs/neo#17171's Decision, as #1518 is).

Live latest-open + closed sweep: the latest 20 open Brain issues, plus open and closed searches for `lint-guides`, "guide lint", "guides: true", `dead-script-ref` and "Engine guides pin", at 2026-11-22T03:14Z. Hits read: #1518 (the source), #1891 (no guide-lint scope; its PR shares two files with this one), and the closed #1360, #1366, #1368 and #1783 (whose Out of Scope defers a cross-repository link lint). No equivalent. Backlog: 162 open (red). This is not a fold: it moves scope out of #1518 and adds none.
A2A: no overlapping `[lane-claim]` or `[lane-intent]` in the latest 30 messages at 03:14Z.
MC sweep: 3 `query_raw_memories` queries (problem and system nouns) and 1 `query_messages` query. The only hit is the 2026-11-08 claim for #1518; no prior decision against this.
Own-assignment sweep: 5 open. #1518 is the source, and #1506 (one-shot scripts) does not cover this script.
Structure map: `npm run ai:structure-map -- --files --loc` at `d6f43108` (exit 0). The change deletes one file in `ai/scripts/lint` and adds none.

Owner: @neo-opus-grace, as #1518's owner, so this ticket carries no `good first issue` label.

Origin Session ID: fcc79e4e-58d3-45ee-a307-579657aa8948
Retrieval Hint: "Brain guides: true neo-agent-skills-guides Engine guides neo.mjs pin dead-script-ref ai:fleet-server lint-guides.mjs retire isEntryModule"


## Timeline

- 2026-11-22T03:18:12Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-22T03:18:14Z @neo-opus-grace added the `enhancement` label
- 2026-11-22T03:18:15Z @neo-opus-grace added the `ai` label
- 2026-11-22T03:18:15Z @neo-opus-grace added the `build` label
- 2026-11-22T03:18:37Z @neo-opus-grace marked this issue as being blocked by #1891
- 2026-11-22T03:20:15Z @neo-opus-grace cross-referenced by #1518
- 2026-11-22T03:32:19Z @neo-opus-grace removed the block by #1891
- 2026-11-22T03:33:07Z @neo-opus-grace cross-referenced by PR #1894
- 2026-11-22T05:35:29Z @tobiu referenced in commit `ee5eff6` - "feat(ci): the Brain lints its guides and the Engine's refs to its scripts (#1893) (#1894)

The Brain's caller of the shared baseline passes `guides: true`, so the Brain's own guides are
judged inside the required `PR baseline` context.

`ai:lint-engine-guides` runs the same bin over the Engine's guides at the `neo.mjs` pin, with the
Brain's manifests as script roots, as a Brain Lint step. A Brain change that drops an `ai:*` script
an Engine guide names now fails the pull request that made it. `package-lock.json` joins the
workflow's paths, because it resolves the pin and the bin.

The unwired copy of the guide lint retires. `isEntryModule.spec.mjs` launches
`lint-identity-vocabulary.mjs` through the symlinked checkout instead.

spec-retired: test/playwright/unit/ai/scripts/lint/lintGuides.spec.mjs retires with its subject, ai/scripts/lint/lint-guides.mjs; the same rules live in the neo-agent-skills bin `neo-agent-skills-guides`, whose spec is that repository's scripts/test-check-guides.mjs."
- 2026-11-22T05:35:31Z @tobiu closed this issue

