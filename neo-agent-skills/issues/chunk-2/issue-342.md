---
id: 342
title: The substrate-size guard reds a tree whose AGENTS.md is generated
state: CLOSED
labels:
  - bug
  - ai
  - build
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T03:01:00Z'
updatedAt: '2026-11-11T03:47:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/342'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-11-11T03:47:11Z'
---
# The substrate-size guard reds a tree whose AGENTS.md is generated

## Context

neomjs/neo#20749, the Engine half of #335, untracks the root `AGENTS.md` and generates it at install and on every Start. Its PR baseline reds at `Substrate size`:

```
➖ AGENTS.md                      : not present — N/A for this repository
❌ .claude/CLAUDE.md              : ENOENT: no such file or directory, stat '/home/runner/work/neo/neo/.claude/CLAUDE.md'
check-substrate-size: .claude/CLAUDE.md could not be measured — ENOENT: no such file or directory, stat '/home/runner/work/neo/neo/.claude/CLAUDE.md'
```

The baseline job never installs the caller, so the tracked link `.claude/CLAUDE.md → ../AGENTS.md` names a file that tree has not generated.

## The Problem

The guard reads a dangling entry point as a seat whose rules resolve to nothing. That reading is right when the link names a file the repository tracks. In #335's decided shape, though, the harness entry stays tracked and the file it names is git-ignored and generated at install, so every CI tree that skips the install holds a link to nothing. Today the guard reds that shape, and every consumer that adopts it.

## The Architectural Reality

Design authority: "`AGENTS.md` is untracked and ignored. `.claude/CLAUDE.md` stays the tracked symlink it already is." (#335, the design read 6784477727, which the author's answer 6784580742 accepted as shape C)

- `scripts/check-substrate-size.mjs` `collectReport` decides presence by `lstat` and fails closed on a link `resolveLoadedSize` cannot resolve. Its JSDoc names the case it exists for: a dangling entry would otherwise "go quiet on a seat whose rules resolve to nothing".
- The budget for a generated file is held where the file is made: `generate-agents-md.mjs:266` refuses a variant over the same `PER_FILE_LIMIT_BYTES` (23,800 B). A generated `AGENTS.md` cannot reach a seat over budget, whatever the caller's CI measures.
- `git check-ignore` answers whether a path is ignored, and it never reports a tracked path as ignored.

## The Fix

In `collectReport`, an entry point that is a symlink whose target is absent, inside the tree and ignored by git is an unbuilt projection. The row reads `applicable: false`, with `projection` naming the target, and prints `links to <target>, a git-ignored projection this tree has not generated — N/A here`. Ignore status comes from an injectable `isIgnored` seam, defaulting to `git check-ignore --quiet`. Any answer other than "ignored" (a tracked target, no git, not a repository) keeps today's error. Version bump.

## Acceptance Criteria

- [ ] A link to an absent target that git ignores reports N/A with the target named, and the run exits 0.
- [ ] A link to an absent target that git does not ignore stays an error, both inside a git tree and outside one.
- [ ] A link to an ignored target that exists is measured as before.
- [ ] `npm test` and `npm run lint` pass, and `package.json` carries the bump.

## Out of Scope

- An entry that `@`-imports an ignored target; no consumer spells its entry that way.
- Measuring the projection by generating it here: the generator already refuses an over-budget variant.
- Moving the Engine's baseline tag to the release; neomjs/neo#20749 does that.

## Related

- #335 (the shape), neomjs/neo#20749 (the red run), neomjs/neo#20748

Live latest-open + closed sweep: checked the latest open at 2026-11-11T03:02Z (#336 and #335 nearest, neither on the guard) and a closed search ("substrate-size dangling"); no equivalent. Backlog band: 32 open (orange); no owning ticket covers the guard.
A2A in-flight claim sweep: the last 30 messages (all read states) carry no claim on `check-substrate-size`.
Memory Core rationale sweep: the dangling-entry rule's rationale is its own JSDoc (fail closed for seats); nothing reverses it, and this keeps it for every tracked target.
Own-assignment sweep: #335 is mine and is the shape this serves; #289 and #258 do not touch the guard.
Structure map: N/A. This is the Skills repository, and the owner is `scripts/check-substrate-size.mjs`; no file is added.

Ownership: self-assigned (Grace); it blocks neomjs/neo#20749.

Retrieval Hint: "substrate-size guard dangling link git-ignored generated AGENTS.md projection"

Origin Session ID: 40a9201b-5e9e-4845-ab9f-dee27312077d

## Timeline

- 2026-11-11T03:01:00Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T03:01:02Z @neo-opus-grace added the `bug` label
- 2026-11-11T03:01:02Z @neo-opus-grace added the `ai` label
- 2026-11-11T03:01:02Z @neo-opus-grace added the `build` label
- 2026-11-11T03:01:03Z @neo-opus-grace added the `model-experience` label
- 2026-11-11T03:01:03Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T03:05:34Z @neo-opus-grace cross-referenced by PR #343
- 2026-11-11T03:05:51Z @neo-opus-grace cross-referenced by PR #20749
- 2026-11-11T03:07:35Z @neo-opus-grace cross-referenced by #335
### @neo-opus-grace - 2026-11-11T03:47:11Z

Not planned: neomjs/neo#20749 took the tracked-pointer shape, so no consumer links into an ignored projection. Details on #343.

- 2026-11-11T03:47:12Z @neo-opus-grace closed this issue
- 2026-11-11T03:47:54Z @neo-opus-grace cross-referenced by #20748

