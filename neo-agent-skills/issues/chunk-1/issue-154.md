---
id: 154
title: Dependabot's skills bumps merge themselves on green required checks
state: OPEN
labels:
  - enhancement
  - developer-experience
  - ai
  - build
assignees: []
createdAt: '2026-10-10T00:29:29Z'
updatedAt: '2026-10-10T00:38:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/154'
author: neo-fable-clio
commentsCount: 1
parentIssue: 14
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
# Dependabot's skills bumps merge themselves on green required checks

## Context

Every Skills release reaches each consumer as a standalone Dependabot pull request on its next daily run (#144; the four consumers exclude `neo-agent-skills` from their npm group for exactly this). Each of those PRs is a two-line pin move that waits for the operator's merge, under §critical_gates 1 (human-only `gh pr merge`). The operator, 2026-10-10 00:2xZ: on these PRs the human merge gate can be weakened — peers merging on green CI, or better, auto-merge inside the workflow, "less work for us". The written proposal that followed (the machine merges under a declared predicate; agents still never merge) got his go at 00:3xZ.

Design authority: the operator's go of 2026-10-10 on the proposal of 00:24Z — a repository's own workflow may merge a Dependabot pull request for the `neo-agent-skills` dependency once the required checks pass; no agent runs `gh pr merge`. This narrows §critical_gates 1's blanket human-only scope for that one class of PR and nothing else.

## The Problem

A Skills release costs the operator four merges a day that carry no decision: the pin moves, the consumer's baseline re-runs at the new version, and the only question — did the consumer stay green — is answered by CI. Meanwhile the drift the pins accumulate while they wait is real: on 2026-10-09 the consumers sat at 0.1.30 while 0.1.32 shipped two new skills; a seat loads what its checkout pins. Giving peers merge authority instead would open a new trust surface (seat PATs merging into `dev`) and ask for judgment where a predicate suffices.

## The Architectural Reality

- No consumer allows auto-merge today (`allow_auto_merge: false` on neo, neo-agent-institution, neo-agent-brain, devindex, neo-agent-skills) and none has branch protection on `dev` (required status checks `[]`, no required reviews; read 2026-10-10 00:2xZ). GitHub's auto-merge waits for *required* checks and refuses where the repository does not allow it — so without the operator's per-repo settings `gh pr merge --auto` fails closed, which is the right default.
- The consumers already call one reusable workflow by tag (`reusable-pr-baseline.yml@vX.Y.Z`, Institution `shared-pr-baseline.yml:30`) with caller-owned triggers and caller-owned permissions; the callee holds no consumer credential. A second reusable workflow fits the same shape.
- `dependabot/fetch-metadata` exposes the PR's `dependency-names` and `update-type`; the PR's **author** (`pull_request.user.login == 'dependabot[bot]'`, `type == 'Bot'`) is the identity — never `github.actor`, which is whoever reruns or merges (#148/#149's lesson).
- neo pins the package exactly (`0.1.30`), the other three with a caret; Dependabot moves both shapes. The actions-ecosystem bump of the reusable workflow's tag (`neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml@vX`) is a second PR of the same class (devindex #66 today).
- The gate's text lives in this package: `agents-md/sections/0401-critical-gate-1.md`, rendered into every consumer's AGENTS.md by `generate-agents-md`. Its `mechanical_guard` field reads "none; discipline-only until guard exists".
- #80 (Vega) rejected a different shape — a workflow in this repository pushing pins into consumers with `contents: write` on each of them. A reusable workflow is the opposite: it runs in the consumer, with the consumer's own token and the permissions the consumer declares.

## The Fix

1. **`.github/workflows/reusable-dependabot-automerge.yml`** (this repository, `workflow_call`):
   - inputs: `dependency_allow_list` (default: `neo-agent-skills`, `neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml`), `update_types` (default `patch,minor`), `merge_method` (default `squash`);
   - job `eligibility`: the PR author is `dependabot[bot]`/`Bot`; every dependency the PR names is on the allow-list (`dependabot/fetch-metadata`); the update type is allowed; the repository variable `NEO_AUTOMERGE_SKILLS` is not `off` (the kill switch); anything else exits without touching the PR and says why;
   - job `enable`: `gh pr merge --auto --<method>` with `GH_TOKEN: ${{ github.token }}` — GitHub performs the merge only when every **required** check on the current head passes, and re-waits when Dependabot rebases; a repository without "Allow auto-merge" or without required checks makes this step fail red, never merge.
2. **The callers** (one ~15-line workflow per consumer, filed as a leaf per repository after this ships): `on: pull_request` (opened, synchronize, reopened), `permissions: contents: write, pull-requests: write`, `uses: neomjs/neo-agent-skills/.github/workflows/reusable-dependabot-automerge.yml@vX.Y.Z`.
3. **The operator's admin part, per consumer** (named here, done by him): "Allow auto-merge" on; branch protection on `dev` with the required status checks named (the Shared PR Baseline jobs; neo's unit CI). Until then the workflow fails closed.
4. **The gate's text** (`0401-critical-gate-1.md`): a `machine_merge` line — *a repository's auto-merge workflow merges a Dependabot pull request whose dependencies are on its allow-list once the required checks pass (operator-configured per repository, Skills #N); no agent runs `gh pr merge`, and no approval signal changes that* — and `mechanical_guard` updated to name it for that class.

Slot rule: the gate line is always-loaded substrate (+~220 B in every consumer's AGENTS.md); `MACHINE-ENFORCEABLE` by construction — the workflow is the guard; disposition `keep`.

Decision Record impact: none (release/merge policy under the operator's authority; no ADR governs the merge gate). Design authority above.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `reusable-dependabot-automerge.yml` eligibility | the PR author via the event payload; `dependabot/fetch-metadata` | author `dependabot[bot]`/`Bot` AND all dependencies on the allow-list AND update type allowed AND kill switch not `off` | any miss → exit 0 with a reason line, no mutation; an author lookalike (branch name, title) never qualifies | the workflow header | contract test with fixture events (bot, lookalike, mixed-dependency PR, major bump, switch off) |
| the `enable` step | GitHub auto-merge semantics | `gh pr merge --auto --squash` with the caller's token; the merge happens on green required checks, head-bound | repo without "Allow auto-merge"/required checks → the step fails red, nothing merges | the workflow header + README *Authority* | a consumer dry run before protection (red, no merge) and after (merge on green) |
| the callers (4) | #14's caller-obligation pattern | caller-owned trigger + write permissions + the tag pin | a caller without `contents: write` fails at startup, as the baseline does | each caller's header | per-repo leaf + first auto-merged PR link |
| `0401-critical-gate-1.md` | the operator's design authority | the `machine_merge` line; `mechanical_guard` names the workflow for this class | — | AGENTS.md (generated) | `generate-agents-md` contract; `check-substrate-size` on a consumer |

## Acceptance Criteria

- [ ] AC-1 The reusable workflow exists with the inputs above; `eligibility` passes a fixture Dependabot PR for `neo-agent-skills` (patch) and refuses: a non-bot author with a `dependabot/` branch name, a PR naming a second dependency outside the allow-list, a major bump, and `NEO_AUTOMERGE_SKILLS=off` — each with its reason line (contract test beside `scripts/test-*.mjs`).
- [ ] AC-2 `enable` calls `gh pr merge --auto` with the caller's token only; the workflow declares no secrets and no `pull_request_target`; a static test asserts both.
- [ ] AC-3 The gate's section carries the `machine_merge` line and the updated `mechanical_guard`; `generate-agents-md`'s contract passes; the always-loaded delta is stated in the PR body.
- [ ] AC-4 README *Authority* names the class and the operator's two settings per consumer.
- [ ] AC-5 (post-merge, per consumer) The caller leaf is merged, the operator's settings are on, and the first Dependabot skills PR merges itself on green — its link is the receipt; a consumer without the settings shows the red, un-merged run first (the fail-closed witness).

## Out of Scope

Auto-merging any other dependency (the allow-list is the boundary; widening it is its own decision); peers merging anything; Dependabot PRs in this repository (they validate without publishing per #148 and still wait for the operator unless added to the allow-list later); the `neo` exact-pin shape (Dependabot moves it; unchanged).

## Avoided Traps

- `github.actor` as the identity (a human rerun or merge is the actor; the author is the bot) — #148/#149.
- A workflow in this repository with write access to consumers (#80's rejected shape); the reusable workflow runs with the consumer's token.
- `pull_request_target` with a checkout of the PR head (code execution with write permissions); nothing here checks out the PR.
- Merging on "all checks green" computed by the workflow itself — a second readiness predicate (#76's class); GitHub's required-checks auto-merge is the predicate, head-bound.
- A silent no-op where the settings are missing: the step must go red so the operator sees the missing switch.

## Related

#14 (PR governance across the repositories; the caller obligations), #144 (the Dependabot path), #148/#149 (Dependabot validation-only; the author predicate), #80 (the rejected push-from-Skills shape), #76 (approval survives the push), #140 (integration close pattern); devindex #66 (an actions-ecosystem bump of the reusable tag, open now).

Live latest-open sweep: the latest 20 open issues of this repository, created-descending, at 2026-10-10 00:3xZ (newest #152); no equivalent. Closed-state sweep ("auto-merge OR automerge OR dependabot"): #148/#117/#114/#88 — none is this. A2A in-flight claim sweep at 00:3xZ (12 newest, all read-states): none on this scope. Memory Core rationale sweep: Vega's 2026-09-16 rejection of the push-from-Skills shape (#80) and Grace's 2026-09-24/25 Dependabot facts — both consistent with this shape. Own-assignment sweep: none on this surface. Structure map: N/A (this package; sibling precedent `reusable-pr-baseline.yml`). Meta-skill sweep: the gate section is agents-md substrate; the always-loaded delta is declared above.

unowned-rationale: the operator's admin settings come first; the workflow is a small build open to any seat (Euclid wrote the sibling author predicate); the design seat stays with the author.

Origin Session ID: 357bcb1f-3338-4a2e-b6e0-0e99eea44ccb

Retrieval Hint: "Dependabot auto-merge reusable workflow allow-list neo-agent-skills required checks machine_merge gate line"

## Timeline

- 2026-10-10T00:29:30Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T00:29:30Z @neo-fable-clio added the `developer-experience` label
- 2026-10-10T00:29:31Z @neo-fable-clio added the `ai` label
- 2026-10-10T00:29:31Z @neo-fable-clio added the `build` label
- 2026-10-10T00:29:36Z @neo-fable-clio added parent issue #14
### @neo-fable-clio - 2026-10-10T00:38:39Z

## The operator's part, measured (2026-10-10 00:4xZ)

**Done:** "Allow auto-merge" is on in all five repositories (`allow_auto_merge: true` on neo, neo-agent-institution, neo-agent-brain, devindex, neo-agent-skills).

**Still needed — one ruleset per consumer with "Require status checks to pass".** The default rules on `dev` are deletion + non-fast-forward only (neo adds `code_scanning` and `copilot_code_review`; devindex has none): no repository carries a `required_status_checks` rule (`GET /repos/{r}/rules/branches/dev`). GitHub's auto-merge "merges a pull request automatically after all required reviews and status checks pass", and the option exists "only on pull requests that cannot be merged immediately" — with no required checks a Dependabot PR is mergeable at once, so auto-merge has nothing to wait for and would merge before CI (or refuse to enable). The workflow's `enable` step therefore goes red until the rule exists — the fail-closed witness of AC-5.

The check names to require (exactly as the check runs are named on the last merged PR of each repository):

| Repository | Require (the Shared PR Baseline set + the repository's own gate) |
|---|---|
| neo-agent-institution | `Shared PR Baseline / PR base`, `… / PR body`, `… / Release ref`, `… / Skills version`, `… / Skills materialized`, `… / Substrate size`, `… / Secrets`, `… / Commit authorship`, `… / Source comment archaeology`, `… / npm overrides` · `Isolated Institution` · `Explicit Brain contract` |
| neo-agent-brain | the same ten `Shared PR Baseline / …` · `unit` · `lint` · `substrate` · `suite (head)` · `integration-unified` · `integration-parity` · `check` |
| devindex | the same ten `Shared PR Baseline / …` · `test` |
| neo | `baseline / PR base`, `baseline / PR body`, `baseline / Release ref`, `baseline / Skills version`, `baseline / Skills materialized`, `baseline / Substrate size`, `baseline / Secrets`, `baseline / Commit authorship`, `baseline / Source comment archaeology`, `baseline / npm overrides` · `Classify test scope / Classify test scope` · `build-all` · `check` · `check-size` · `check-freshness` · `components (1/3)` `(2/3)` `(3/3)` |

Two consequences to decide with the rule: it binds every PR, not only Dependabot's (a human merge with a red required check becomes impossible unless the operator is on the ruleset's bypass list); and a check that is skipped by path filters reads as "expected but missing" and blocks — the names above all ran on ordinary PRs, which is why they are the candidates.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 357bcb1f-3338-4a2e-b6e0-0e99eea44ccb


