---
id: 220
title: The commit-authorship test lets the seat's GIT_* environment win
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - testing
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-26T18:58:31Z'
updatedAt: '2026-10-27T20:27:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/220'
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
closedAt: '2026-10-27T20:27:52Z'
---
# The commit-authorship test lets the seat's GIT_* environment win

## Context

`scripts/test-commit-authorship.mjs` fails on any machine whose environment carries a git identity, and passes in CI, which carries none. Agent seats carry their git identity in the launch environment. Three seats from three model families have hit it: defect-notes on 2026-10-17 and 2026-10-23, and a third run on 2026-10-26. Each time, the assertion reports the seat's own address where it expects `operator@home.example`.

The red also hides other runners. `npm test` chains its runners with `&&` (`#37`), and this is the 12th of 15, so on such a machine `test-tag-release`, `test-check-version-bump` and `test-release-origin` never run.

## The Problem

The test's header promises isolation: the operator's identity comes from a stand-in global config, and `NEO_AGENT_IDENTITY` is cleared. Line 26 then builds the children's environment as `{...process.env, GIT_CONFIG_GLOBAL, GIT_CONFIG_NOSYSTEM, NEO_AGENT_IDENTITY: ''}`. Three inherited channels outrank the stand-in config file, and each was measured on `dev` at `1ab34a7`:

| inherited variables | effect | measured |
|---|---|---|
| `GIT_AUTHOR_*` / `GIT_COMMITTER_*` | outrank `user.*` from every config file | exit 1, actual `'leak@example.com'` |
| `GIT_CONFIG_COUNT`, `GIT_CONFIG_KEY_n`, `GIT_CONFIG_VALUE_n` | config entries above every file | exit 1, actual `'leak@example.com'` |
| `GIT_CONFIG_PARAMETERS` (what `git -c` hands its children) | the same | exit 1, actual `'leak2@example.com'` |

Reproduce:

```bash
GIT_AUTHOR_EMAIL=leak@example.com GIT_COMMITTER_EMAIL=leak@example.com node scripts/test-commit-authorship.mjs
```

## The Architectural Reality

- `scripts/test-commit-authorship.mjs:26` defines the one `env` that every `git` child and every spawn of the guard receives. The only other child, at line 268, runs a pure parse and no git.
- The guard, `scripts/check-commit-authorship.mjs`, is not at fault. It reads git's resolved identity, and that is what it must read.
- Other runners spread `process.env` too, `test-tag-release.mjs` and `test-release-origin.mjs` among them, but none asserts an identity.

## The Fix

Build the children's environment from `process.env` minus every `GIT_`-prefixed key, then add the test's own three:

```js
env = {...Object.fromEntries(Object.entries(process.env).filter(([key]) => !key.startsWith('GIT_'))),
       GIT_CONFIG_GLOBAL: join(root, 'global.gitconfig'), GIT_CONFIG_NOSYSTEM: '1', NEO_AGENT_IDENTITY: ''},
```

Then extend the header's isolation sentence to say that inherited `GIT_*` variables are dropped. Dropping only the four identity keys is not enough, because the two config channels carry `user.email` past it. A scratch copy patched this way exits 0 with all three channels set at once, and the unpatched copy in the same layout exits 1.

`check-version-bump` requires a new version on every non-Dependabot pull request, even though this file is not published. So bump both `package.json` and `package-lock.json` with `npm version patch --no-git-tag-version`.

## Acceptance Criteria

- [ ] The reproduce command above exits 0. It exits 1 on `dev` first.
- [ ] `GIT_CONFIG_COUNT=1 GIT_CONFIG_KEY_0=user.email GIT_CONFIG_VALUE_0=leak@example.com node scripts/test-commit-authorship.mjs` exits 0. It exits 1 on `dev` first.
- [ ] With none of these variables set, the runner still ends `check-commit-authorship: contract arms green.`
- [ ] Besides the version bump, only `scripts/test-commit-authorship.mjs` changes: line 26 and the header comment.

## Out of Scope

- The `&&` chain in `npm test` that hides the later runners (`#37`).
- The other runners that spread `process.env`. None of them asserts an identity.

## Related

- #37: `npm test` short-circuits, which turns this red into three unexecuted runners.
- #93 and #109: the guard and its rebase arm, whose contract this test checks.

## Sweeps

Live latest-open sweep: the latest 20 open issues in this repository, read at 2026-10-26T18:57:34Z. None is equivalent; `#37` is the related `&&`-chain defect.
A2A claim sweep: the last 30 messages in all read-states, back to 17:08Z. No claim on this file.
MC sweep: one query on the symptom ("test-commit-authorship fails on a seat whose environment sets GIT_AUTHOR_EMAIL"), 10 results. It surfaced the 2026-10-17 defect-note and no prior decision.
Own-assignment sweep: 1 open (`#215`), not overlapping.

Structure-map gate: N/A. One existing file under `scripts/` changes, and nothing is placed.
Decision Record impact: none.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: c39fbf61-9160-44c8-ba56-9545b1b50774

Authored by 🖖 Grace (Claude Opus 5.5, Claude Code) · session c39fbf61-9160-44c8-ba56-9545b1b50774


## Timeline

- 2026-10-26T18:58:32Z @neo-opus-grace added the `bug` label
- 2026-10-26T18:58:32Z @neo-opus-grace added the `good first issue` label
- 2026-10-26T18:58:32Z @neo-opus-grace added the `ai` label
- 2026-10-26T18:58:33Z @neo-opus-grace added the `testing` label
- 2026-10-27T20:08:18Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-27T20:10:19Z @neo-gemini-maryam cross-referenced by PR #223
- 2026-10-27T20:27:52Z @tobiu referenced in commit `0d5351a` - "fix(testing): strip inherited GIT_* variables in test-commit-authorship (#220) (#223)"
- 2026-10-27T20:27:52Z @tobiu closed this issue

