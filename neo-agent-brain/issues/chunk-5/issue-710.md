---
id: 710
title: 'A seat''s repository records its forge, and a GitLab slug may name nested groups'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T17:52:14Z'
updatedAt: '2026-10-01T21:06:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/710'
author: neo-opus-grace
commentsCount: 0
parentIssue: 684
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 712 A seat''s PAT is presented only to the forge host it was stored for'
closedAt: '2026-10-01T21:06:18Z'
---
# A seat's repository records its forge, and a GitLab slug may name nested groups

## Context

Leaf 1 of #684 (the Fleet's GitLab parity), split in the #684 intake (comment 5933517917) and amended with @neo-opus-ada's three points (15:03Z). It lands on the shared repository rule that #683 merged (17:26Z).

A GitLab seat cannot be registered today: a seat's repository is a two-segment `owner/repo`, its clone URL defaults to GitHub, and a self-hosted host does not say which forge it is.

## The Problem

- `FleetManager.repoCoordinates` (the one rule `setRepo` and `setRepos` share) calls `assertRepoSlug`, which accepts exactly two segments (`deriveAgentRepoPath.mjs:83-84`). A GitLab project in nested groups (`group/sub/project`) is refused.
- Without a clone URL, a repository defaults to `https://github.com/<slug>.git`. That is right for GitHub, and a silent wrong host for anything else.
- `setRepos` refuses a duplicate by comparing slug strings. That is enough while every checkout is `<seat>/<owner>/<repo>`, but nested slugs break it in two ways: `acme/tools/cli` would clone inside `acme/tools`, and a GitHub and a GitLab `acme/tools` would share one directory.
- No field records the forge, and the two later leaves (clone auth, injection) need it.

## The Architectural Reality

- `ai/services/fleet/FleetManager.mjs`: `repoCoordinates` (`:25-45`), `setRepo` (`:443`), `setRepos` (`:468-500`).
- `ai/services/fleet/deriveAgentRepoPath.mjs`: `RESERVED_OWNERS` (`:31`, `harness` plus #681's `memory`), `deriveAgentRepoPath` (`:64`), `assertRepoSlug` (`:83`), `assertSeatSegment`, `assertContained`.
- `src/fleet/contract/wire.mjs` lists method names only, so a `forge` field crosses the wire unchanged.

## The Fix

1. A repository entry carries `forge`, `'github'` by default or `'gitlab'`, recorded on `metadata.repo` and on each `metadata.repos[]` entry.
2. A GitLab slug may have more than two segments. Each one passes `assertSeatSegment`, segment 0 is checked against `RESERVED_OWNERS`, and the checkout path mirrors the segments under `<root>/<agentId>/`. A GitHub slug stays `owner/repo`.
3. A GitLab entry needs an explicit clone URL naming its path; only GitHub keeps the default.
4. `setRepo` and `setRepos` refuse a seat set whose derived checkout paths collide, meaning one equals or contains another, across the working repository and the extras and across forges.

## Contract Ledger

*Backfilled 2026-10-01 by the author, for RA-2 of @neo-gpt's review of PR #711. Clone authentication (#712) and the GitLab workflow server's injection (#727) are separate leaves with their own ledgers.*

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `setRepo({id, repoSlug, cloneUrl, forge})` / `setRepos({id, repos: [{repoSlug, cloneUrl, forge}]})` registration fields | `FleetManager.repoCoordinates` | `forge` is `'github'` (default) or `'gitlab'`; any other value refuses | omitted: GitHub | `setRepo` / `setRepos` JSDoc | `FleetManager.spec` GitLab arm |
| `metadata.repo` / `metadata.repos[]` serialization | the registry row | GitHub entries keep today's `{repoSlug, cloneUrl}`, with no `forge` written; GitLab entries add `forge: 'gitlab'`; readers treat an absent forge as GitHub | absent: GitHub | same | same |
| Per-forge depth | `repoCoordinates` | GitHub: exactly `<owner>/<repo>`. GitLab: two or more segments (nested groups) | none (refusal) | same | same |
| Clone URL | `repoCoordinates` | A plain string with no query, fragment or whitespace, naming the slug as an https, ssh or SCP-like remote with no credentials. GitLab must name one; GitHub defaults to `https://github.com/<slug>.git`. Refusals never echo the URL | GitHub default only | `repoCoordinates` JSDoc | `FleetManager.spec` (query, fragment, array and path-mismatch controls) |
| Checkout path helper | `deriveAgentRepoPath` / `assertRepoSlug` | Any validated depth of two segments or more: forge-independent path math, under `<root>/<agentId>/`; segment 0 never a `RESERVED_OWNERS` key | none (refusal) | module JSDoc | `deriveAgentRepoPath.spec` |
| Checkout collisions | `assertNoCheckoutCollision` | Across the working repository and the extras, on either forge, no derived path equals or contains another; refused before any registry write | none (refusal) | function JSDoc | `FleetManager.spec` collision arm |

## Acceptance Criteria

- [ ] AC-1: an entry records its forge (`github` by default); an unknown forge refuses.
- [ ] AC-2: a GitLab slug with nested groups registers, and its checkout path mirrors the groups; a reserved first segment refuses.
- [ ] AC-3: a GitLab entry without a clone URL refuses; a GitLab clone URL must name the same path.
- [ ] AC-4: a nested, equal or cross-forge path collision refuses in `setRepo` and in `setRepos`, and nothing is written.
- [ ] AC-5: every existing GitHub arm passes unchanged.

## Out of Scope

- Clone authentication for the forge host (leaf 2).
- `NEO_GITLAB_*` injection, the plan and rendering (leaf 3).
- The cockpit form (`neomjs/neo-agent-institution#245`).

## Decision Record impact

`none`.

## Related

Parent #684 · #683 / #682 (`setRepos` and the shared rule) · #681 (`RESERVED_OWNERS`) · #704 / #706 (the seat-home record, which keys the seat home, not repository paths)

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T17:51:48Z. No equivalent; #684 is the parent.
- Exact: "forge gitlab slug nested" finds nothing open.
- A2A: the 15 newest, all states. My own 17:51Z announcement; no competing claim.
- MC: the #684 intake trail, with Ada's amendments folded in.
- Own-assignment: #684 (the parent).

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "Fleet repository forge github gitlab nested group slug checkout path containment setRepos repoCoordinates"

🖖 Grace (Claude Opus 5.5, Claude Code)



## Timeline

- 2026-10-01T17:52:16Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T17:52:16Z @neo-opus-grace added the `ai` label
- 2026-10-01T17:52:16Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T17:52:17Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T17:52:26Z @neo-opus-grace added parent issue #684
- 2026-10-01T17:57:28Z @neo-opus-grace cross-referenced by PR #711
- 2026-10-01T18:10:28Z @neo-opus-grace cross-referenced by #712
- 2026-10-01T18:10:42Z @neo-opus-grace marked this issue as blocking #712
- 2026-10-01T18:24:47Z @neo-opus-grace cross-referenced by #684
- 2026-10-01T18:41:35Z @neo-opus-ada cross-referenced by #407
- 2026-10-01T18:54:48Z @neo-opus-ada cross-referenced by #721
- 2026-10-01T19:31:34Z @neo-opus-grace cross-referenced by #725
- 2026-10-01T19:49:32Z @neo-opus-grace referenced in commit `100c535` - "feat(fleet): a seat's repository records its forge, and a GitLab slug may name nested groups (#710)

repoCoordinates takes forge (github default, never written; gitlab recorded): GitHub stays exactly owner/repo with its github.com default, GitLab may name nested groups and must name its clone URL. Path derivation accepts any validated depth, so callers are unchanged. setRepo and setRepos refuse a seat set whose checkouts would share a path or nest, across forges."
- 2026-10-01T20:10:26Z @neo-opus-grace cross-referenced by #727
- 2026-10-01T20:23:48Z @neo-opus-grace cross-referenced by #729
- 2026-10-01T20:32:13Z @neo-opus-grace referenced in commit `92414aa` - "fix(fleet): a repository's clone URL is a plain remote string, so a query or fragment cannot stand in for its path (#710)

Euclid's RA-1 on PR #711: the remote matcher read query and fragment text as part of the host, and RegExp.test coerced an array. repoCoordinates now refuses a non-string or a URL carrying a query, fragment or whitespace before matching."
- 2026-10-01T21:06:18Z @tobiu referenced in commit `425667d` - "feat(fleet): a seat's repository records its forge, and a GitLab slug may name nested groups (#710) (#711)

* feat(fleet): a seat's repository records its forge, and a GitLab slug may name nested groups (#710)

repoCoordinates takes forge (github default, never written; gitlab recorded): GitHub stays exactly owner/repo with its github.com default, GitLab may name nested groups and must name its clone URL. Path derivation accepts any validated depth, so callers are unchanged. setRepo and setRepos refuse a seat set whose checkouts would share a path or nest, across forges.

* fix(fleet): a repository's clone URL is a plain remote string, so a query or fragment cannot stand in for its path (#710)

Euclid's RA-1 on PR #711: the remote matcher read query and fragment text as part of the host, and RegExp.test coerced an array. repoCoordinates now refuses a non-string or a URL carrying a query, fragment or whitespace before matching."
- 2026-10-01T21:06:18Z @tobiu closed this issue

