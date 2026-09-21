---
id: 140
title: 'The shared baseline is the last consumer still on 0.1.5, one release after the engine moved'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-09-15T15:05:15Z'
updatedAt: '2026-09-15T15:22:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/140'
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
closedAt: '2026-09-15T15:22:22Z'
---
# The shared baseline is the last consumer still on 0.1.5, one release after the engine moved

## Context

`#138` moved this repository's three Skills coordinates to 0.1.5 earlier today. `neo-agent-skills@0.1.6` shipped since — it teaches the archaeology guard that a typed `[not-ticket-ref: <reason>]` marker binds to a **reference**, not only to a colour ([neo-agent-skills#72](https://github.com/neomjs/neo-agent-skills/pull/72)). `neomjs/neo` consumed it in [#18742](https://github.com/neomjs/neo/pull/18742), merged `e1f1d00307`, and retired its own bare `ticket-ref-ok` marker in the same change.

This repository is now the only consumer left behind.

Live sweeps 2026-09-15T15:0xZ: no open PRs, no competing A2A claim, full-text over open and closed for `skills 0.1.6 pin bump` returns nothing.

## The Problem

```
shared-pr-baseline.yml  uses: …@27e151d9   (0.1.5)
package.json            "neo-agent-skills": "^0.1.5"
package-lock.json       resolves            0.1.5
registry latest                             0.1.6
```

The consequence is the same divergence `#138` existed to end, one release later: a durable comment here carrying `#1234 [not-ticket-ref: authority]` fails this repository's baseline while the identical line passes in `neomjs/neo`. **Two consumers of one governance substrate disagreeing about the same rule** is what `neomjs/neo#16553` was opened to stop, and what `#18742` just finished stopping on the engine side.

**All three coordinates move, because two different jobs read two different ones** — measured on `#139` and again on `#18742`:

| job | resolution | pin |
|---|---|---|
| `skills-materialized` | `npx --no-install …-materialize --check` — the **caller's** `node_modules` | the **lockfile** |
| `source-comment-archaeology`, `substrate-size`, `pr-body` | `npm install --prefix "$RUNNER_TEMP" neo-agent-skills@${SKILLS_VERSION}` | the **`uses:` SHA** |

## The Fix

Move `uses:` to `@6b009521fac6f571be2200cadbeab734cfcffe6d`, the merge commit of `#72`; the dependency floor to `^0.1.6`; and the lockfile with it. Update the adoption comment so the recorded coordinate matches.

Nothing else. 0.1.6 only **widens** what the guard accepts, so it cannot newly fail a line 0.1.5 passed.

## Acceptance Criteria

- [ ] **AC-1** — `shared-pr-baseline.yml` pins `@6b009521fac6f571be2200cadbeab734cfcffe6d`.
- [ ] **AC-2** — that coordinate carries `SKILLS_VERSION: '0.1.6'` at all three call sites, read from the Skills repository at that SHA rather than assumed.
- [ ] **AC-3** — `package-lock.json` resolves 0.1.6, so `skills-materialized` and the isolated-install jobs agree on one release.
- [ ] **AC-4** — the shared baseline is green on the PR. Its diff touches no `.mjs`, so the archaeology green is **vacuous** and must be labelled as such; the non-vacuous evidence is a positive control against a base whose diff carries `.mjs`.

## Out of Scope

- **Three pre-existing bare `ticket-ref-ok` markers** — `apps/agentos/model/WakeSignal.mjs`, `harness/preload.cjs`, `resources/scss/src/apps/agentos/Viewport.scss`. Only the first is in the guard's scope at all, and `invalid-escape` on the legacy marker predates 0.1.6 — 0.1.5 rejects it too. Not caused by this bump.
- **Two pre-existing full-audit violations** carrying refs with no marker: `test/playwright/unit/apps/agentos/view/fleet/memories/rowComponents.spec.mjs:24` and `.../roster/cardIdentity.spec.mjs:93`. CI never sees them because the baseline selects changed files only; the first PR that edits either reds on prose it did not write. Recorded so the next reader does not re-measure.
- Retiring the bare marker here as `#18742` did in the engine. This repository runs no second implementation, so there is nothing to converge — it is a comment cleanup, not a divergence repair, and it needs its own ticket.

## Avoided Traps

- **Do not move only the SHA.** `skills-materialized` reads the lockfile and would stay on the old materializer.
- **Do not read a permissive range as a current install.** `^0.1.5` accepts 0.1.6 and the lockfile pins 0.1.5. The falsifier is `node -p "require('./node_modules/neo-agent-skills/package.json').version"`.
- **Do not read this PR's archaeology green as coverage.** No `.mjs` in the diff means zero files measured.

## Decision Record impact

none.

## Related

[neo-agent-skills#72](https://github.com/neomjs/neo-agent-skills/pull/72) (0.1.6) · [neomjs/neo#18742](https://github.com/neomjs/neo/pull/18742) (the engine consumer) · `neomjs/neo#18740` · `#138` / `#139` (the 0.1.5 move here)

Origin Session ID: 148d12cc-9777-46e4-bfb9-a422479153a5

Retrieval Hint: `query_raw_memories("institution shared baseline 0.1.6 pin move last consumer")`

## Timeline

- 2026-09-15T15:05:16Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-15T15:05:17Z @neo-opus-grace added the `enhancement` label
- 2026-09-15T15:05:17Z @neo-opus-grace added the `agent-os` label
- 2026-09-15T15:05:17Z @neo-opus-grace added the `ai` label
- 2026-09-15T15:05:18Z @neo-opus-grace added the `dependencies` label
- 2026-09-15T15:06:30Z @neo-opus-grace cross-referenced by PR #141
- 2026-09-15T15:22:22Z @tobiu referenced in commit `7963d24` - "Merge pull request #141 from neomjs/grace/140-skills-016

chore(ci): the last consumer moves to Skills 0.1.6 (#140)"
- 2026-09-15T15:22:22Z @tobiu closed this issue

