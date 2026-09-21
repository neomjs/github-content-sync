---
id: 93
title: 'The commit-authorship guard ships as neo-agent-skills-commit-authorship, with the team roster as its caller''s input'
state: CLOSED
labels:
  - enhancement
assignees:
  - neo-opus-ada
createdAt: '2026-09-19T15:40:20Z'
updatedAt: '2026-09-19T17:16:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/93'
author: neo-opus-ada
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
closedAt: '2026-09-19T17:16:05Z'
---
# The commit-authorship guard ships as neo-agent-skills-commit-authorship, with the team roster as its caller's input

## Context

The Skills half of neomjs/neo#18757. `neo-agent-brain:ai/scripts/lint/check-commit-authorship.mjs` refuses agent commits that carry the operator's identity or credit an address outside the team, and nothing runs it. `neomjs/neo` cannot call it (no Brain dependency), and CI has no caller. This package already ships guards the same way (`neo-agent-skills-secrets`, #91).

## The constraint that shapes it

@tobiu, 2026-09-19: the roster marks **our team**, whose agents are equal peers with their own GitHub accounts and addresses, apart from **everyone else**: other projects running the Agent OS with their own PAT logins, forks, and contributors who are not maintainers. A roster shipped inside this package would impose our team on every other deployment. So the package ships the guard's **logic**, and the **roster is its caller's input**.

The guard's two checks already split along that line:

| check | roster | caller |
|---|---|---|
| operator identity: a commit from an agent checkout authored with the global git identity | none — it compares against `git config --global user.email`, deliberately | pre-push hook, any deployment |
| `Co-Authored-By` trailers on agent-authored commits that name no seat's address | the team boundary: whose commits are agent-authored, which addresses are seats | hook and CI (`--author-login`) |

## The Fix

- `scripts/check-commit-authorship.mjs`, ported from the Brain guard with its behaviour kept: ref-tuple ranges, linked-worktree or `NEO_AGENT_IDENTITY` ownership, the authenticated agent lane, fail-open on unreadable input only, and agent-authored offenders blocking while others stay advisory.
- `--roster <module>`: a module exporting `registryAgentLogins()` and `rosterEmailForLogin(login)`. The Brain's `ai/graph/agentCoAuthorEmails.mjs` already satisfies that contract. The team's domains are the domains its seats' addresses use. **With no roster, the trailer check inspects nothing**, and the operator check still runs.
- `--base <sha>` for CI, where there is no pre-push payload: the range is `<base>..HEAD`.
- Bin `neo-agent-skills-commit-authorship`, a contract test, and the release that carries it.
- A `reusable-pr-baseline.yml` job that runs the pinned guard with `--author-login` from the GitHub-authenticated PR author. Its roster comes from caller inputs naming a repository, ref and path, checked out apart from the PR. A PR can therefore never add itself to the roster that judges it. With no roster input, the job has nothing to check and says so.

## Acceptance Criteria

- [ ] Hook mode: from a linked worktree, a pushed commit authored with the global identity is refused, and the same push from the main checkout passes.
- [ ] With a roster: an agent-authored commit whose trailer names an address outside the roster is refused. The same trailer on a non-agent commit only warns, and a seat's address passes.
- [ ] With no roster: the trailer check reports nothing, the operator check still runs, and the output says the roster is absent.
- [ ] `--author-login` for a roster login marks the lane as agent even when the commit's own `%ae` claims otherwise, which is the forged-author case.
- [ ] CI job: red for a non-compliant fixture and green for a compliant one, with the roster read from a ref other than the PR head.

## Out of Scope

- Wiring `neomjs/neo`'s `.husky/pre-push` and its caller inputs (neomjs/neo#18757).
- Retiring the Brain's copy of the guard, and moving `findUnknownCoAuthors` out of `agentCoAuthorEmails.mjs` (a Brain follow-up once this ships).
- A commit-time installer for other repositories (#90).

## Related

neomjs/neo#18757 · #14 (PR governance epic) · #91 (the same shape for the credential guard) · neomjs/neo#17195 (the author-scoped boundary)

Live latest-open sweep: the open issues in this repository at 2026-09-19T15:40Z. #90 is adjacent (installers), and nothing is equivalent.

Origin Session ID: 6ecb7b5f-dc26-48a3-8e49-7232159377c1
Retrieval Hint: "commit authorship guard publish roster input team boundary author-login reusable baseline"

## Timeline

- 2026-09-19T15:50:26Z @neo-opus-ada cross-referenced by PR #94

