---
id: 353
title: Consume neo-agent-skills 0.1.7 — the Brain's lockfile has held 0.1.3 through four releases
state: CLOSED
labels:
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-09-15T16:37:25Z'
updatedAt: '2026-09-15T17:03:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/353'
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
closedAt: '2026-09-15T17:03:16Z'
---
# Consume neo-agent-skills 0.1.7 — the Brain's lockfile has held 0.1.3 through four releases

## Context

`#294` moved this repository to `neo-agent-skills@0.1.3` on adoption. Four releases have shipped since — 0.1.4, 0.1.5, 0.1.6, 0.1.7 — and the lockfile has held 0.1.3 throughout.

Live sweeps 2026-09-15T16:3xZ: latest 6 open read; full-text over open and closed for `neo-agent-skills version bump` returns `#294` (closed, the adoption) and `#352` (open, @neo-opus-vega's dependabot automation — adjacent, not this). One open PR in this repo, `#351`, mine, unrelated in content.

## The Problem

```
package.json      "neo-agent-skills": "^0.1.3"
package-lock.json  node_modules/neo-agent-skills → 0.1.3
installed                                          0.1.3
registry latest                                    0.1.7
```

**A caret does not float once a lockfile exists.** The range permits 0.1.7 and the lock freezes 0.1.3, so every seat installing from this repository gets 0.1.3 deterministically — stale rather than variable.

**And this repository is a consumer, which is the part I got wrong first.** I swept the org earlier today and recorded the Brain as *"range only, not a baseline consumer"*. True of the reusable PR baseline; **false of skills tooling**, which is the half that matters:

- `package.json:59` — `"postinstall": "neo-agent-skills-materialize"`
- `.github/workflows/substrate-sync.yml:42` — `npx --no-install neo-agent-skills-materialize --check`

So every agent working in this repository has been reading a skill corpus four releases old, and CI has been asserting that corpus is current against a materializer four releases old. Corrected after @neo-opus-vega refused the claim and measured it.

## What the four releases carry

The corpus projected into `.claude/skills` is the payload here, not the guard binaries — this repository calls no reusable baseline. Between 0.1.3 and 0.1.7 that corpus gained, among other things, the archaeology rules for colour literals, leading zeros and numeric HTML entities, the typed reference escape, and 0.1.7's premise pre-flight row — *"the premise pre-flight asks the inverse question, so a diff cannot quietly reverse what was ratified"*, which is review discipline this repository's agents are meant to be running.

## The Fix

Move the dependency floor to `^0.1.7` and refresh the lockfile. Nothing else.

**The caret stays.** Pin *shape* is a live policy question @neo-opus-vega is holding on `#352` and deliberately scoped out of her dependabot lane; changing it here would fork that decision from under her. This ticket moves the version, not the convention.

## Acceptance Criteria

- [ ] **AC-1** — `package.json` declares `^0.1.7` and `package-lock.json` resolves 0.1.7.
- [ ] **AC-2** — `npx --no-install neo-agent-skills-materialize --check` exits 0 and reports `v0.1.7`, which is the check `substrate-sync.yml` runs.
- [ ] **AC-3** — no tracked file changes beyond those two. `.claude/skills/` is a gitignored projection; if the newer materializer wants to commit bytes, that is a finding rather than an expected diff.
- [ ] **AC-4** — `package.json`'s 4-space formatting is preserved; npm must not reserialize the file.

## Out of Scope

- **Dependabot and version-update automation** — `#352`, @neo-opus-vega's, operator-instructed. This ticket is the one-time catch-up that automation would otherwise raise as four separate PRs; it does not replace the automation.
- **Caret vs exact pin as a convention.** Held on `#352`.
- The reusable PR baseline. This repository does not call it, which is exactly the distinction that made me mis-sweep it.

## Avoided Traps

- **Do not read `^0.1.3` as "resolves to latest".** It does not while a lockfile exists. The falsifier is `node -p "require('./node_modules/neo-agent-skills/package.json').version"`, never the range.
- **Do not let `npm install` reserialize `package.json`.** Verify the diff is one line before committing.
- **Do not assume the projection is inert.** Run the `--check` the workflow runs; a materializer four releases newer is the one thing here that could legitimately change output.

## Decision Record impact

none.

## Related

`#294` (the 0.1.3 adoption this supersedes) · `#352` (dependabot automation, adjacent) · `neomjs/neo-agent-institution#140` (the same catch-up, same shape) · `neomjs/neo-agent-skills#14`

Origin Session ID: 148d12cc-9777-46e4-bfb9-a422479153a5

Retrieval Hint: `query_raw_memories("brain lockfile held 0.1.3 four releases caret does not float")`

## Timeline

- 2026-09-15T16:37:25Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-15T16:37:26Z @neo-opus-grace added the `dependencies` label
- 2026-09-15T16:38:06Z @neo-opus-grace cross-referenced by PR #354
- 2026-09-15T16:42:33Z @neo-opus-vega cross-referenced by PR #355
- 2026-09-15T17:03:16Z @tobiu referenced in commit `37cb5df` - "Merge pull request #354 from neomjs/grace/brain-skills-017

chore(deps): neo-agent-skills 0.1.7, after four releases held at 0.1.3 (#353)"
- 2026-09-15T17:03:17Z @tobiu closed this issue

