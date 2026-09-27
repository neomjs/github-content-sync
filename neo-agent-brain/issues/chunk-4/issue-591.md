---
id: 591
title: 'A seat''s first Start clones its repo with the seat''s own PAT, not the host''s credentials'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T15:46:19Z'
updatedAt: '2026-09-27T16:41:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/591'
author: neo-opus-ada
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-27T16:41:17Z'
---
# A seat's first Start clones its repo with the seat's own PAT, not the host's credentials

## Context

FM onboarding: an agent added in FM holds its GitHub PAT (#577), and its first Start clones the seat's working repo (#589 / neomjs/neo-agent-institution#297). Emmy traced the clone on dev `d5cd907` (A2A, 15:41Z). `provisionAgentRepo` runs `git clone -- <url> <path>` with the Fleet process's own environment, and the seat's PAT reaches only the later harness spawn. So a private repo clones only if the Fleet host already holds Git credentials that can read it. That is shell setup, and the FM cannot own it.

## The Fix

The seat's first Start clones with the seat's own PAT:
- `startAgentProvisioned` hands its resolved credential to the repo step.
- The default clone executor presents it only to `https://github.com` (the PAT's own host). It uses a host-scoped credential helper that reads the token from the child's environment, so the token never appears in argv. The inherited helpers are reset, so the host's ambient credentials are not tried first, and a prompt is never opened.
- Any other remote, or a clone without a credential, runs as before.

## Acceptance Criteria

- [ ] AC-1 A GitHub https clone with a seat credential runs with a host-scoped helper, the token only in the child's env and nowhere in argv. Any other remote, or no credential, gets a plain clone (spec over the command builder).
- [ ] AC-2 `startAgentProvisioned` passes the resolved credential to the repo step (spec).
- [ ] AC-3 A real `git` run against a private neomjs repo with a bogus seat token fails on authentication, while the same command without the helper succeeds through the host's credentials. This shows the seat's token is presented and the ambient credentials are not (recorded in the PR).

Parent: #571

Live latest-open sweep (2026-09-27T15:50Z): the open issues and PRs of this repository show none on the clone's credentials. #589 / #590 is the clone URL's boundary, not its authentication.

Authored by Ada (Claude Opus 5.5, Claude Code). Session f3d50317-fe3b-4773-b4ac-db05e1fa6812.


## Timeline

- 2026-09-27T15:46:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T15:46:21Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T15:46:22Z @neo-opus-ada added the `ai` label
- 2026-09-27T15:46:24Z @neo-opus-ada added parent issue #571
- 2026-09-27T15:50:52Z @neo-opus-ada cross-referenced by PR #592
- 2026-09-27T16:21:46Z @neo-opus-ada referenced in commit `112962d` - "fix(fleet): a seat's clone runs outside the host's Git setup, so no ambient rewrite, header or askpass reaches it (#591)

Euclid's action on f3495ab: resetting credential.helper left the rest of the
host's Git setup live, so a global url.insteadOf, http.extraHeader or askpass
could redirect or authenticate the seat's clone. The seat branch of
gitCloneCommand now runs with GIT_CONFIG_GLOBAL=/dev/null and
GIT_CONFIG_NOSYSTEM, and drops GIT_ASKPASS, SSH_ASKPASS and every
environment-injected config variable; proxy and CA settings still reach it
through the environment. provisionAgentRepoGit.spec drives a real git through
a poisoned HOME and a recording proxy: before this change the clone was
rewritten to the local server, after it one CONNECT to github.com is all that
leaves."
- 2026-09-27T16:24:40Z @neo-opus-ada referenced in commit `6400c92` - "fix(fleet): a seat's clone has no home directory, so a host ~/.netrc cannot authenticate it (#591)

curl retries a 401 with a matching ~/.netrc entry before git's credential flow
runs, so a host netrc for github.com would have cloned as the host, not the
seat. The seat branch now sets HOME to the null device. The new Git arm shows
the host's git sending its netrc entry over plain http and the seat clone's
environment sending none (red before this change)."
- 2026-09-27T16:41:17Z @tobiu referenced in commit `35303f1` - "Merge pull request #592 from neomjs/ada/591-seat-clone-pat

feat(fleet): a seat's first Start clones its repo with the seat's own PAT, not the host's credentials (#591)"
- 2026-09-27T16:41:18Z @tobiu closed this issue

