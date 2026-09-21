---
id: 369
title: 'The hook-CI parity guard was copied to neo, not moved — the Brain copy is orphaned'
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
  - tech-debt
assignees:
  - neo-opus-vega
createdAt: '2026-09-18T10:53:31Z'
updatedAt: '2026-09-18T12:13:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/369'
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
closedAt: '2026-09-18T12:13:01Z'
---
# The hook-CI parity guard was copied to neo, not moved — the Brain copy is orphaned

## Context

`neomjs/neo@2bd8aabce0` (2026-09-16, `neomjs/neo#18753` resolving `neomjs/neo#17783`) is titled *"the hook-CI parity guard moves to the repo it governs and gains its first caller"*. It is **additive only** — 5 files, 1074 insertions, **0 deletions**. It created the guard in `neomjs/neo` and never removed the one here, so both exist and only one runs.

Found while discharging `@neo-opus-ada`'s required action on `PR #365`: the `@summary` there cites a census of neo's guards, and re-measuring it (3 → 4 → **10 of 15**) surfaced that the relocation the same docblock says the work *"lands with"* had already happened. `PR #365` is closed unmerged and `#364` closed as superseded; this is their residue.

## The Problem

`ai/scripts/lint/lint-guard-ci-parity.mjs` checks that every guard a repository runs in a pre-commit hook is also run by a CI workflow — a guard a `--no-verify` commit would otherwise slip past.

**This repository has no hooks.** There is no `.husky/` directory and `package.json` has no `lint-staged` block (it is `null`). The guard's subject does not exist here, independently of whether anything invokes it.

Nothing does. Enumerated on `origin/dev`:

| candidate executor | present |
|---|---|
| a step in any of the 10 `.github/workflows/*.yml` | **no** — zero matches for `guard-ci-parity` |
| a `lint-staged` entry | **no** — the block is `null` |
| a husky hook | **no** — there is no `.husky/` |
| an npm script | **no** |
| `test/playwright/unit/ai/scripts/lint/lintGuardCiParity.spec.mjs` | **yes — the only caller** |

So ~460 lines of guard, a registry, and a ~500-line spec exist to test each other. The spec also carries five pre-existing local reds (the guard's own self-tree state), which is noise every future reader has to classify before discovering none of it runs.

Nothing is red today, and that is the point: `lint-script-plane.mjs` discovers workflow entrypoints dynamically through `readWorkflowEntrypoints()`, so an orphan simply stops being found rather than being reported. `script-plane-lint` is green on `dev` at `64be6ca0a`. The only stale assertion is prose — `ai/scripts/lint/lint-script-plane.mjs:245` still offers `run: node ./ai/scripts/lint/lint-guard-ci-parity.mjs` as its worked example of a workflow-invoked script, which is no longer true of any workflow here.

## The Architectural Reality

- **Live successor:** `neomjs/neo` carries `buildScripts/util/check-guard-ci-parity.mjs`, `buildScripts/util/check-guard-ci-parity-registry.json`, `test/playwright/unit/buildScripts/util/check-guard-ci-parity.spec.mjs` and `.github/workflows/guard-ci-parity-lint.yml`. That repo has the `.husky/` hooks and the `lint-staged` block the guard reads, so it is where the check can mean anything.
- **The correction this PR was going to make is already there.** At `buildScripts/util/check-guard-ci-parity.mjs:258` the live copy rejects `paths-ignore` and deliberately accepts a `paths` allowlist — `#364`'s fix, shipped independently.
- **`lint-script-plane.mjs` is unaffected mechanically.** Its `readWorkflowEntrypoints()` parses workflow YAML at run time; removing an unreferenced module changes nothing it computes. Only its docblock names the file.
- This is the Accretion Defense case in `AGENTS.md §self_evolving_systems`: the diff is net-negative, and the layer beneath the one being removed (a hook plane) does not exist in this repository.

## The Fix

Delete, in this repository:

- `ai/scripts/lint/lint-guard-ci-parity.mjs`
- `ai/scripts/lint/guard-ci-parity-registry.json`
- `test/playwright/unit/ai/scripts/lint/lintGuardCiParity.spec.mjs`

and replace the stale invocation example in `ai/scripts/lint/lint-script-plane.mjs`'s `readWorkflowEntrypoints` `@summary` with a `run:` line a workflow in this repository actually contains.

No behaviour changes: nothing executes the deleted module, and the docblock edit is a comment.

## Decision Record impact

`none`. The relocation's authority is `neomjs/neo#17783`; this removes a leftover on the losing side of a move that already happened.

## Acceptance Criteria

- [ ] The three files are deleted, and `grep -rn "guard-ci-parity" --include='*.mjs' --include='*.json' --include='*.yml' .` (excluding `node_modules`) returns only the corrected `lint-script-plane.mjs` docblock, or nothing.
- [ ] `lint-script-plane.mjs`'s `readWorkflowEntrypoints` `@summary` cites a `run:` line that exists in a `.github/workflows/*.yml` file in this repository, quoted from it.
- [ ] `script-plane-lint`, `substrate`, `unit`, `lint` and both integration jobs are green on the PR head — the deletion removes a module nothing reads, so no job may change verdict.
- [ ] The Brain unit suite's failure **set** on the branch is a subset of the merge base's, by failure title, and the five `lintGuardCiParity` arms are in the difference. No title appears on the branch that is absent from the base.
- [ ] The diff touches no file outside this repository, so `neomjs/neo`'s copy is untouched by construction — checked against the PR's own changed-file list, not against neo's CI.

*(AC-4 corrected 2026-09-18, after measuring: the count form is not satisfiable in this suite. Two full runs of the same tree differ by roughly a dozen titles — base 88 failed, branch 72, while only 5 of the 16 in the difference are the deleted spec's. The rest are run-to-run variance in the same environment, which the `did not run` count corroborates (61 vs 60). A count subtraction would have been reported as a clean −5 or panicked at −16; the set comparison answers the question the AC was actually asking, which is whether the branch introduces a red. See the defect-note on the local/CI divergence.)*

*(AC-5 corrected 2026-09-18, before the PR opened: it first asked for `neomjs/neo`'s `guard-ci-parity-lint` to be green on neo `dev` after this merges. That is unobservable — the workflow is `pull_request`-triggered with a `paths` filter, so it never runs on a `dev` merge, and a PR in this repository cannot trigger anything in neo regardless. An AC whose evidence cannot exist is worse than no AC, because a reviewer can tick it.)*

## Out of Scope

- **Any change to `neomjs/neo`'s copy.** The live guard, its registry, its workflow and the `scan ⊆ paths` alignment predicate are that repository's concern and are tracked there.
- **Re-introducing hooks to this repository.** If the Brain ever gains a hook plane, the guard is one `npm install` away in `neo-agent-skills`; restoring a deleted file is cheaper than carrying a dead one.
- **Auditing the other `ai/scripts/lint/*` modules for the same orphaning.** A real question, and a separate sweep — filing it here would make this deletion wait on an audit.

## Avoided Traps

- **Deleting the spec's five reds as if they were this ticket's defects.** They are the guard's own self-tree state and they leave with the file; the AC compares failure *titles* against the merge base so their disappearance is accounted for rather than celebrated.
- **Assuming `lint-script-plane.mjs` hardcodes the path and would red.** It does not — it parses workflow YAML at run time. Checked before writing the AC.
- **Treating "the commit says moves" as evidence the Brain copy is dead.** The commit is additive; the evidence is the absent hook plane and the zero invocations, not the commit subject.

---

**Live latest-open sweep:** checked the latest 20 open issues in this repository at 2026-09-18T10:53Z; no equivalent found (`#362` is Brain-tier membership duplication, `#246` is content-root hardwiring — neither covers this module). A repo-wide `guard-ci-parity` issue search returns only `#364`, now closed.
**A2A in-flight sweep:** `list_messages({status:'all', limit:30})` at the same time; the recent claims are `neomjs/neo-agent-institution#149`/`#150`/`#151`, `neomjs/neo#18837`–`#18842` and `neo-agent-skills#88`/`#89` — none touches `ai/scripts/lint/`.
**Own-assignment sweep:** `#362`, `#237`, `#23`, `#64`, `#65` — embedding lane, tenant ingestion and Brain-tier membership; no overlap.

Retrieval Hint: "orphaned lint-guard-ci-parity in the Brain after the neo relocation, no hooks to check parity against"

Origin Session ID: 2baab8f1-a2b2-4183-b903-8d7231796cab




## Timeline

- 2026-09-18T10:53:31Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-18T10:53:33Z @neo-opus-vega added the `enhancement` label
- 2026-09-18T10:53:33Z @neo-opus-vega added the `ai` label
- 2026-09-18T10:53:33Z @neo-opus-vega added the `refactoring` label
- 2026-09-18T10:53:33Z @neo-opus-vega added the `agent-os` label
- 2026-09-18T10:53:33Z @neo-opus-vega added the `tech-debt` label
- 2026-09-18T10:53:45Z @neo-opus-vega cross-referenced by PR #365
- 2026-09-18T10:53:46Z @neo-opus-vega cross-referenced by #364
- 2026-09-18T11:41:46Z @neo-opus-vega cross-referenced by PR #370
- 2026-09-18T12:01:44Z @neo-opus-vega referenced in commit `decd81b` - "chore(lint): the import-safe note states its reason, not a false universal (#369)

The previous wording claimed every lint in ai/scripts/lint is import-safe.
Four are not: check-commit-authorship.mjs, explicitGraphStoreOwnership.mjs,
lint-pr-stacking.mjs and prStackingGuard.mjs have no `process.argv[1]`
guard, and check-commit-authorship.mjs ends in a bare process.exit(1) at
module scope.

Caught by @neo-opus-grace in review. The attribution it replaced named one
file and was checkable; the universal was asserted from a positive sample
and was wrong. The reason stands without either."
- 2026-09-18T12:13:01Z @tobiu closed this issue
- 2026-09-18T12:13:02Z @tobiu referenced in commit `d5ae3e8` - "Merge pull request #370 from neomjs/agent/369-delete-orphaned-parity-guard

chore(lint): the hook-CI parity guard leaves the repo that cannot run it (#369)"

