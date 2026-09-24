---
id: 112
title: 'materialize-harness-skills links a symlinked --root into dangling paths: the target is physical, the link directory is not'
state: CLOSED
labels:
  - bug
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T17:41:46Z'
updatedAt: '2026-09-24T20:12:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/112'
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
closedAt: '2026-09-24T20:12:30Z'
---
# materialize-harness-skills links a symlinked --root into dangling paths: the target is physical, the link directory is not

## Context

`test-lint-skill-corpus.mjs` fails one case on macOS: "a clean non-Neo consumer resolves projected document references". It failed on every run on `dev@793b580` and on #111's branch, apart from one pass. `dev`'s CI, which runs on Linux, is green. Every seat runs its local gates on macOS, so every skills PR shows a red that isn't its own. #111 spent six probes proving that.

## The Problem

`scripts/materialize-harness-skills.mjs` builds each link with `relative()`, from two different path spaces:
- **The target, physical:** `skillsIn` comes from `packageRoot = resolve(here, '..')`, and `here` is the entry module's URL, which Node's loader resolves to its real path.
- **The link's directory, as given:** `facadeDir` and `agentsDir` come from `root = resolve(args.root || consumerRoot())`, which keeps the path it was given.

The affected lines are `targetFor` (L136) and the `.agents/skills` link (L147 in `--check`, L251 at creation).

**Mechanism, measured on macOS:**
- `os.tmpdir()` returns `/var/folders/…`, a logical path; `/var` is a symlink to `/private/var`. `process.cwd()` in the same directory returns `/private/var/folders/…`.
- The fixture passes `--root` under `tmpdir()`, so every link becomes `../../../../../../../../private/var/folders/…/node_modules/neo-agent-skills/.agents/skills/<name>`.
- The OS resolves that from the link's physical directory, one level deeper, and lands in `/private/private/var/…`.
- `--check` then reports `.claude/skills/<name> is a dangling link; its node_modules target is gone` for all 37 skills.

I reproduced it by hand with a `mkdtemp` consumer under `$TMPDIR`. A root derived from the cwd is physical and safe. Any `--root` given through a symlinked path breaks, on every OS.

## The Fix

Resolve the root physically, so both sides of every `relative()` are in one path space:

```js
root = realpathSync(resolve(args.root || consumerRoot()))
```

## Acceptance Criteria

- [ ] **AC-1:** With `--root` given through a symlinked path, every materialized link resolves and `--check` exits 0.
- [ ] **AC-2:** A regression arm creates the symlinked root explicitly: a real consumer directory plus a symlink to it, passed as `--root`. The arm then reds on Linux CI without the fix, rather than relying on macOS's `/var`.
- [ ] **AC-3:** `test-lint-skill-corpus.mjs` passes locally on macOS at the fix.
- [ ] **AC-4:** The release ships in the PR: `package.json`, the lockfile and every `SKILLS_VERSION` pin.

## Out of Scope

- The render-verify documentation (#111).
- Any change to how a consumer installs or pins the package.

## Related

#111 (where it surfaced) · #66

Sweeps at 17:41Z:
- **Latest open:** the latest 20 open issues (17:30Z); none equivalent.
- **Keywords:** `realpath`, `dangling`, `materialize link`, `symlink`, `tmpdir`; no equivalent, open or closed.
- **MC:** the dangling-link and realpath query matched only the April skill-sync issue, which is a different defect.
- **A2A:** no claim.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T17:41:47Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T17:41:48Z @neo-opus-grace added the `bug` label
- 2026-09-24T17:41:48Z @neo-opus-grace added the `build` label
- 2026-09-24T17:45:34Z @neo-opus-grace cross-referenced by PR #113
### @neo-opus-grace - 2026-09-24T17:59:27Z

## Contract Ledger (T3), for PR #113

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `neo-agent-skills-materialize` (`scripts/materialize-harness-skills.mjs`): `--root` resolution | #112 | The root is realpathed before any link is computed, so each link is relative between two physical paths. This matches `packageRoot`, which Node realpaths. | A `--root` that does not exist now throws `ENOENT` at startup. Before, the first write failed. | No. One source comment at the line. | `test-lint-skill-corpus`, new case "a consumer reached through a symlinked root resolves every projected link". Red-first at a physical `TMPDIR` (the Linux condition): 25/26. With the fix: 26/26 in both `TMPDIR` modes. |
| The install path (`INIT_CWD` or cwd-derived root) | unchanged | Unchanged: `process.cwd()` is already physical, measured as `/private/var/…` beside `os.tmpdir()`'s `/var/…`. | — | No | The existing "clean non-Neo consumer" case stays green. |
| Release 0.1.17: `package.json`, the lockfile and all six `SKILLS_VERSION` pins, as set by #111 | The skills release rule (operator, on #72); one publish after #113 (operator, 2026-09-24) | A consumer that moves to the pinned release gets both fixes. | — | No | `test-reusable-pr-baseline` is green. |

🖖 Grace (Claude Opus 5.5, Claude Code) · session 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2


- 2026-09-24T18:04:16Z @neo-gpt cross-referenced by PR #111
- 2026-09-24T19:21:30Z @neo-opus-grace referenced in commit `b7ffd76` - "fix(materializer): links are computed from a physical root, so a symlinked --root no longer dangles (#112)

Node realpaths the running package, so skillsIn was physical while the root stayed as given; relative links across the two path spaces dangled whenever the root sat behind a symlink, which on macOS includes every os.tmpdir() consumer. The root is realpathed. A consumer case reaches the materializer through a symlink one level shallower than the checkout, red on any OS without the fix. Release 0.1.17: package.json, the lockfile and all six SKILLS_VERSION pins."
- 2026-09-24T19:21:30Z @neo-opus-grace referenced in commit `f6912c4` - "chore(release): the materializer fix ships as 0.1.18, since #111 took 0.1.17 (#112)

Rebased onto dev after #111 merged; package.json, the lockfile and all six SKILLS_VERSION pins move to 0.1.18."
- 2026-09-24T20:12:30Z @tobiu referenced in commit `f7b29e7` - "Merge pull request #113 from neomjs/grace/112-materializer-realpath

fix(materializer): links are computed from a physical root, so a symlinked --root no longer dangles (#112)"
- 2026-09-24T20:12:30Z @tobiu closed this issue

