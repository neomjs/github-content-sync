---
id: 367
title: 'The skill scripts break on a Windows checkout (CRLF, directory links)'
state: CLOSED
labels:
  - bug
  - windows
  - contributor-experience
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T01:40:41Z'
updatedAt: '2026-11-12T02:16:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/367'
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
closedAt: '2026-11-12T02:16:56Z'
---
# The skill scripts break on a Windows checkout (CRLF, directory links)

## Context

@vivaangarg2005, a first-time contributor working on Windows, opened #355 for #353. It bundled the #353 fix with portability changes for running this package's scripts on Windows. Their #357 then narrowed to #353 alone and merged (`a0ff7eb`).

Maryam's triage held #355 as superseded for #353 and noted that the Windows changes had no ticket (comment `IC_kwDOUEM8o88AAAABlfRmsw` on #355). Under the operator's "Yes, maintainers fix" ruling (#358), maintainers file the ticket and repair the PR instead of asking the contributor to. This is that ticket, for work #355 already carries.

## The Problem

A Windows checkout gets CRLF text files by default (`core.autocrlf=true`). On such a checkout:
1. `scripts/generate-agents-md.mjs#parseSection` matches front matter with `\n` only, so a CRLF section throws "missing front-matter block".
2. `scripts/lint-skill-corpus.mjs` measures budgets with `statSync().size`. A CRLF file counts one extra byte per line, so it fails budgets that pass on an LF checkout. Its `SKILL.md` front-matter match has the same `\n`-only shape.
3. `scripts/materialize-harness-skills.mjs` creates directory links with `symlinkSync(target, path)`. On Windows a `dir` symlink needs Developer Mode or admin rights; a junction does not. Its `--check` also compares `readlinkSync()` text with the expected relative target, and a junction stores an absolute target, so a correct junction reads as wrong.

## The Architectural Reality

Design authority: `"Checking that a link exists and resolves says nothing about WHERE it resolves."` (the `--check` failure text in `materialize-harness-skills.mjs`). The check's intent is where a link resolves, which a resolved-path comparison states directly and a link-text comparison only approximates. The budgets measure the repository's canonical LF bytes, which is what the package publishes.

- **What #355's diff changes:**
  - `parseSection` takes `\r?\n`;
  - `lint-skill-corpus` counts LF-normalized bytes and normalizes `SKILL.md` before its front-matter match;
  - `materialize` links with `'junction'` on `win32` (`'dir'` elsewhere) and compares `realpathSync` on both sides;
  - the two specs follow.
- **POSIX behavior is unchanged.** An LF file has no `\r\n` to normalize, and a symlink resolves to its target.
- **None of the five files changed on `dev`** since #355's base.

## The Fix

Rescope #355 in place:
1. Merge `dev` into the fork branch. `scripts/agents-md-pointer.mjs` and its spec take `dev`'s version, since the #353 fix landed through #357. `package.json` and `package-lock.json` take `dev`'s, since a fork PR keeps the base version (#358).
2. The PR's title and body name this ticket and credit the contributor.

## Acceptance Criteria

- [ ] #355's diff against `dev` touches only the five Windows files, and `npm run lint` plus `npm test` pass in CI (POSIX).
- [ ] The materialize `--check` still fails a link that resolves to the wrong tree: the mutation arms in `test-materialize-harness-skills.mjs` stay able to go red.

## Out of Scope

- The Brain's Windows setup (`neomjs/neo-agent-brain#1540`).
- A Windows CI runner. The Windows behavior is the contributor's environment; the PR records it if they run it.

## Related

#355, #353 / #357, #358 (fork PR version rule), `neomjs/neo-agent-brain#1540`

Live latest-open + closed sweep: latest 20 open (36 open, orange band; `#366`..`#51`) plus an all-state search (`windows OR CRLF OR junction OR "line endings"`) at 2026-11-12T01:38Z; no ticket on this scope.
A2A in-flight claim sweep: latest messages at 01:30Z; no claim on #355.
MC sweep: `skill scripts fail on a Windows checkout: CRLF front matter, symlink junction, byte budget differs`, 5 results; none on this scope.
Own-assignment sweep: 5 open (#366, #364, #282, #102, #298), none overlapping.
Structure map: N/A (no file placed; the edits are in place).

Ownership: the implementation is @vivaangarg2005's (#355); self-assigned, @neo-opus-ada, for the repair and routing.

Origin Session ID: a84af7d7-9095-4958-8288-503b774a5f6b
Retrieval Hint: "skill scripts break on a Windows checkout: CRLF front matter, LF-normalized budgets, junction directory links, realpath link check — #355 rescope"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T01:40:41Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T01:40:43Z @neo-opus-ada added the `bug` label
- 2026-11-12T01:40:43Z @neo-opus-ada added the `windows` label
- 2026-11-12T01:40:43Z @neo-opus-ada added the `contributor-experience` label
- 2026-11-12T01:40:43Z @neo-opus-ada added the `ai` label
- 2026-11-12T01:49:03Z @neo-opus-ada referenced in commit `3009e89` - "Merge dev into fix-agents-clean-filter: #353 landed via #357, keep the Windows changes (#367)

# Conflicts:
#	package-lock.json
#	package.json
#	scripts/agents-md-pointer.mjs
#	scripts/test-agents-md-pointer.mjs"
- 2026-11-12T01:49:03Z @neo-opus-ada referenced in commit `986327b` - "test(scripts): a link into the wrong skill still fails the materialize check (#367)"
- 2026-11-12T01:49:58Z @neo-opus-ada cross-referenced by PR #355
- 2026-11-12T02:10:26Z @neo-gemini-maryam cross-referenced by #368
- 2026-11-12T02:16:55Z @tobiu referenced in commit `b95e2fe` - "fix(scripts): the skill scripts handle a Windows checkout (#367) (#355)

* fix(agents-md): support worktrees without installed package

Fall back to the pointer held in the index when a worktree lacks the clean-filter module, and cover the shared-config worktree case. Normalize CRLF inputs and make directory-link tests portable on Windows.

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>

* test(scripts): a link into the wrong skill still fails the materialize check (#367)

---------

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
Co-authored-by: Ada <neo-opus-4-7@neomjs.com>"
- 2026-11-12T02:16:57Z @tobiu closed this issue

