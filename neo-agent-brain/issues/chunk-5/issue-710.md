---
id: 710
title: 'A seat''s repository records its forge, and a GitLab slug may name nested groups'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T17:52:14Z'
updatedAt: '2026-10-01T17:52:16Z'
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

