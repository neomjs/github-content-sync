---
id: 353
title: AGENTS.md clean filter crashes git where the package isn't installed
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-11T20:12:20Z'
updatedAt: '2026-11-12T00:53:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/353'
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
closedAt: '2026-11-12T00:53:55Z'
---
# AGENTS.md clean filter crashes git where the package isn't installed

## Context

Since 1.4.0 (#346, PR #347), an install registers the `neo-agents-md` git filter in the consumer's clone: clean is `node node_modules/neo-agent-skills/scripts/agents-md-pointer.mjs --repo <repo>`, smudge is `cat`, and the filter is required. neomjs/neo adopted it in neomjs/neo#20791 (merged 2026-11-11).

Git keeps filter settings per clone, and every worktree of the clone shares them. Git runs the clean command from the top of the worktree it is working in. A worktree whose `node_modules` has no `neo-agent-skills/scripts/agents-md-pointer.mjs` (never installed, or installed below 1.4.0) cannot run it. The filter is required, so git dies as soon as it has to clean `AGENTS.md` there.

Reproduced 2026-11-11 in a neomjs/neo clone (git 2.53.0): `git worktree add --detach <dir> origin/dev`, then `touch AGENTS.md` and `git status`:

```
Error: Cannot find module '<dir>/node_modules/neo-agent-skills/scripts/agents-md-pointer.mjs'
```

The same filter shape in a throwaway repository, for both `git status` and `git add AGENTS.md`:

```
error: external filter 'node node_modules/x/p.mjs' failed
fatal: AGENTS.md: clean filter 't' failed
```

Recovery today: `npm install` in that worktree.

## The Problem

A worktree without the package breaks git once anything rewrites `AGENTS.md` in it. Maintainers keep review worktrees that never install. The error names a missing module, not the cause or the recovery.

## The Architectural Reality

- `scripts/agents-md-pointer.mjs` `cleanCommand(repo)` builds the command above. `ensureFilter()` writes it to `--local` config, so a clone has one clean command for all its worktrees.
- Design authority: the module JSDoc of `scripts/agents-md-pointer.mjs`: "Git keeps only a pointer: `.gitattributes` routes `/AGENTS.md` through the {@link AGENTS_MD_FILTER} filter, whose clean command is this module's CLI and answers any working copy with the pointer." A worktree without the module gets no answer.
- The same JSDoc keeps the filter required on purpose: "a clean command that cannot run makes git refuse to store the file, never store it raw." The fix keeps that: git never stores the generated rules.
- `refresh()` already treats the index entry as the pointer's authority.

## The Fix

`cleanCommand(repo)` returns a POSIX `sh` command. When the worktree has the module, it runs it as today. When the module is absent, it drains stdin and answers the pointer the index already holds:

```sh
if [ -f node_modules/neo-agent-skills/scripts/agents-md-pointer.mjs ]; then node node_modules/neo-agent-skills/scripts/agents-md-pointer.mjs --repo <repo>; else cat >/dev/null; git cat-file blob :AGENTS.md; fi
```

Probed with git 2.53.0 and a 4 MB working copy: `git status` and `git add AGENTS.md` exit 0, nothing is staged, and the stored blob stays the pointer. `ensureFilter()` re-registers a changed command value, so a clone picks the new command up at its next install. Update the module JSDoc and the `cleanCommand` JSDoc, then bump the version.

## Acceptance Criteria

- [ ] `scripts/test-agents-md-pointer.mjs` gains a real-git case: a repository with the registered filter, a worktree or checkout without the module, and a rewritten `AGENTS.md`. `git status` and `git add AGENTS.md` exit 0, and `git cat-file blob :AGENTS.md` is still the pointer. The case fails against the current command.
- [ ] With the module present, the clean command still answers `pointerFor(repo)`; the existing cases pass.
- [ ] The package version is bumped.

## Out of Scope

- neomjs/neo's transitional delegate `buildScripts/util/agentsMdPointer.mjs`, the clean command that commits before neomjs/neo#20791 registered. It retires with neomjs/neo#20790.
- Making the filter optional: git would then store the generated rules.

## Avoided Traps

- An absolute path to the installing checkout's module: removing that worktree would break git in every other one.
- Falling back on any failure (`node … || …`): a broken module would then hide behind a silently stale pointer. The `-f` test falls back only when the module is absent, and a broken module still fails loudly.

## Related

#346 · #347 · neomjs/neo#20788 · neomjs/neo#20791 · neomjs/neo#20790

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 of 35 open (orange band, no owning open ticket: #346 is closed) plus a closed search (`clean filter OR worktree OR agents-md pointer`; #346 is the feature this repairs, no exclusion), at 2026-11-11T20:11Z. A2A in-flight claim sweep: the latest 30 messages, back to 19:20Z, carry no claim on this surface. MC sweep: no sighting beyond my own trail. Own-assignment sweep: #350, #335, #289, #258, no overlap. Structure map: N/A (one existing script, no placement change).

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 15ea444b-707c-4458-bba5-79b4f61fb280

Retrieval Hint: "AGENTS.md clean filter required, worktree without node_modules, git status fatal, fall back to the index's pointer"

## Timeline

- 2026-11-11T20:12:21Z @neo-opus-grace added the `bug` label
- 2026-11-11T20:12:21Z @neo-opus-grace added the `good first issue` label
- 2026-11-11T20:12:21Z @neo-opus-grace added the `ai` label
- 2026-11-11T20:12:22Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T20:12:57Z @neo-opus-grace cross-referenced by #20790
- 2026-11-11T22:43:10Z @neo-gemini-maryam cross-referenced by PR #357
- 2026-11-12T00:22:59Z @neo-opus-ada cross-referenced by #358
- 2026-11-12T00:34:00Z @neo-gemini-maryam referenced in commit `b7dcdf4` - "chore(release): retain base package version on fork PR (#353)"
- 2026-11-12T00:34:35Z @neo-gemini-maryam cross-referenced by PR #355
- 2026-11-12T00:53:54Z @tobiu referenced in commit `a0ff7eb` - "fix(agents-md): clean filter falls back to the index pointer when the module is absent (#353) (#357)

* fix(agents-md): clean filter falls back to the index pointer when the module is absent (#353)

Git keeps filter settings per clone, and worktrees share them. A worktree without neo-agent-skills in node_modules failed whenever AGENTS.md had to be cleaned because the filter was required and the pointer script was missing.

The clean command now checks if the module script exists: when present, it delegates to it as before; when absent, it drains stdin and answers the pointer already stored in the index. Update JSDocs and bump version to 1.4.3.

* chore(release): retain base package version on fork PR (#353)

---------

Co-authored-by: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-12T00:53:56Z @tobiu closed this issue
- 2026-11-12T01:40:42Z @neo-opus-ada cross-referenced by #367
- 2026-11-12T01:49:03Z @neo-opus-ada referenced in commit `3009e89` - "Merge dev into fix-agents-clean-filter: #353 landed via #357, keep the Windows changes (#367)

# Conflicts:
#	package-lock.json
#	package.json
#	scripts/agents-md-pointer.mjs
#	scripts/test-agents-md-pointer.mjs"

