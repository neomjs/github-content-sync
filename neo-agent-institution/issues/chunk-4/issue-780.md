---
id: 780
title: Add claim-responder.yml workflow for curated contributor issues
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - build
  - contributor-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-10-21T21:27:47Z'
updatedAt: '2026-10-21T22:40:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/780'
author: neo-gemini-julia
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
closedAt: '2026-10-21T22:40:06Z'
---
# Add claim-responder.yml workflow for curated contributor issues

## Context

`CONTRIBUTING.md` (`#778` / PR `#779`, merged `2026-10-21`) documents under **3. Claim before you start** (`CONTRIBUTING.md:122`): *"Comment on the ticket that you are taking it. An automatic reply tells you whether you are first in line."*

While `neomjs/neo` runs `.github/workflows/claim-responder.yml` (`neomjs/neo#19202`, expanded in `neomjs/neo#19971` / PR `#19972`), `neomjs/neo-agent-institution/.github/workflows/` currently contains only `agent-pr-body-lint.yml`, `agent-pr-review-body-lint.yml`, `agent-ticket-archaeology-lint.yml`, and `ci.yml`.

Design authority: `CONTRIBUTING.md:122` (`#778` / PR `#779`) and `neomjs/neo#19202`.

## The Problem

An outside contributor who follows `CONTRIBUTING.md:122` and comments a claim on a `good first issue` or `help wanted` ticket in `neomjs/neo-agent-institution` receives no automatic reply confirming whether they are first in line or whether another contributor already holds the ticket.

## The Architectural Reality

- `neomjs/neo/.github/workflows/claim-responder.yml:1-87` triggers on `issue_comment: types: [created]` for open non-PR issues carrying `good first issue` or `help wanted` when the commenter is not a `Bot`, `COLLABORATOR`, `MEMBER`, or `OWNER`.
- All reply logic lives in `neomjs/neo/buildScripts/util/claimResponse.mjs` (`curatedLabel`, `decideReply`, `isClaim`), which accepts parameterized `repoUrl: https://github.com/${owner}/${repo}` and has zero repo-hardcoded dependencies.
- Both `neomjs/neo` and `neomjs/neo-agent-institution` use `dev` as their `defaultBranchRef`, and `.github/workflows/ci.yml:84-88` already uses `actions/checkout@v7` with `repository:` and `ref: dev` for cross-repo checkout. Checking out `buildScripts/util/claimResponse.mjs` from `repository: neomjs/neo` (`ref: dev`, `persist-credentials: false`, `sparse-checkout: buildScripts/util/claimResponse.mjs`, `sparse-checkout-cone-mode: false`) reuses the single source of truth in `neomjs/neo` without duplicating `claimResponse.mjs` across repositories (`AGENTS.md §self_evolving_systems` Accretion Defense).

## The Fix

Add `.github/workflows/claim-responder.yml` to `neomjs/neo-agent-institution`, mirroring `neomjs/neo/.github/workflows/claim-responder.yml` and sparse-checking out `buildScripts/util/claimResponse.mjs` from `neomjs/neo@dev`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.github/workflows/claim-responder.yml` | `CONTRIBUTING.md:122`, `neomjs/neo#19202` | On `issue_comment.created` for open `good first issue` / `help wanted` issues by outside contributors, sparse-check out `neomjs/neo/buildScripts/util/claimResponse.mjs@dev` and post `decideReply(...)` when non-null | Logs `No reply: not a claim, the assignee, or a claimant already answered here.` and exits `0` | `CONTRIBUTING.md:122` | AC-1, AC-2, AC-3 |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] AC-1 [L1]: `.github/workflows/claim-responder.yml` exists in `neomjs/neo-agent-institution`, triggers on `issue_comment: types: [created]` with least-privilege permissions (`contents: read`, `issues: write`, `pull-requests: read`), and sparse-checks out `buildScripts/util/claimResponse.mjs` from `neomjs/neo@dev` with `persist-credentials: false` and `sparse-checkout-cone-mode: false`.
- [ ] AC-2 [L1]: The workflow never interpolates untrusted comment text into shell steps (`actions/github-script@v9` reads `context.payload`), and passes `js-yaml` structural parsing and `neo-agent-skills` baseline guards.
- [ ] AC-3 [L1]: The `Answer the claim` `actions/github-script@v9` step passes `context.repo` (`owner`, `repo`, `repoUrl: https://github.com/${owner}/${repo}`) into `curatedLabel` and `decideReply` and posts the reply via `github.rest.issues.createComment`.

## Out of Scope

- Duplicating `buildScripts/util/claimResponse.mjs` into `neomjs/neo-agent-institution` (owned and unit-tested in `neomjs/neo`).
- Auto-assigning issues to outside claimants (assignment remains a maintainer action per `CONTRIBUTING.md`).

## Avoided Traps

- Copying `claimResponse.mjs` into `neomjs/neo-agent-institution`: creates a second copy that drifts whenever claim patterns are updated in `neomjs/neo` (`#19971` / PR `#19972`).

## Related

- `neomjs/neo-agent-institution#778` / PR `#779` (`CONTRIBUTING.md`)
- `neomjs/neo#19202`, `neomjs/neo#19971` / PR `#19972`

Sweeps:
- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` (`sort: 'created'`) at `2026-10-21T21:25:16Z`; no equivalent found.
- A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at `2026-10-21T21:24:54Z`; no competing claim on this scope.
- Memory Core rationale sweep & KB ticket sweep: queried `claim-responder.yml claimResponse.mjs CONTRIBUTING.md neo-agent-brain neo-agent-institution`; no prior ticket in `neo-agent-institution`.
- Own-assignment sweep: `0` open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-institution`.
- Agent OS structure map (`npm run ai:structure-map -- --files --loc`): `.github/workflows/` owns GitHub Actions workflows (`N/A` for `ai/` modules).

Origin Session ID: fd51794e-dcdf-4240-a11c-75e32cb5e357
Retrieval Hint: "claim-responder.yml neo-agent-institution CONTRIBUTING.md claimResponse.mjs sparse-checkout"


## Timeline

- 2026-10-21T21:27:47Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-21T21:27:49Z @neo-gemini-julia added the `enhancement` label
- 2026-10-21T21:27:49Z @neo-gemini-julia added the `agent-os` label
- 2026-10-21T21:27:49Z @neo-gemini-julia added the `ai` label
- 2026-10-21T21:27:49Z @neo-gemini-julia added the `build` label
- 2026-10-21T21:27:50Z @neo-gemini-julia added the `contributor-experience` label
- 2026-10-21T21:32:12Z @neo-gemini-julia cross-referenced by PR #781
- 2026-10-21T21:58:53Z @neo-gemini-julia referenced in commit `6920f07` - "fix(build): preserve queued claim events and refresh issue state (#780)"
- 2026-10-21T22:40:06Z @tobiu referenced in commit `019d377` - "feat(build): add claim-responder.yml workflow for curated contributor issues (#780) (#781)

* feat(build): add claim-responder.yml workflow for curated contributor issues (#780)

* fix(build): preserve queued claim events and refresh issue state (#780)"
- 2026-10-21T22:40:06Z @tobiu closed this issue

