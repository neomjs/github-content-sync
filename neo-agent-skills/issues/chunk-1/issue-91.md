---
id: 91
title: 'Publish the credential guard, and run it in the shared PR baseline'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - security
assignees:
  - neo-opus-ada
createdAt: '2026-09-18T16:28:34Z'
updatedAt: '2026-09-18T16:54:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/91'
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
closedAt: '2026-09-18T16:54:17Z'
---
# Publish the credential guard, and run it in the shared PR baseline

## Context

@tobiu, 2026-09-18, on neomjs/neo#18917 (merged): *"the CI in general looks like something that we need inside multiple org repos. in this case it could use a FU ticket inside the skills repo."*

neomjs/neo#18832, via PR #18906, gave the engine `buildScripts/util/check-secrets.mjs`:
- It fails on credential-shaped literals (Google API keys and OAuth tokens, OpenAI, Anthropic, GitHub classic and fine-grained, AWS access key ids), each anchored to the credential's real length.
- A finding names file, line and kind, and never the match, because CI logs are published.
- A line that must keep such a literal carries `secret-scan-ok: <reason>`.
- neomjs/neo#18913 (PR #18917) made that reason end at its comment's closer and say something.

The guard reads text files and depends on nothing in the engine. Yet only the engine runs it.

## The Problem

The incident behind neomjs/neo#18832 was a committed key that GitHub's scanner found after the fact. It can happen in any org repository. Measured at 2026-09-18 on each repository's `dev` (`create-app`: `main`), by reading every workflow file, with neo's `secrets-lint.yml` as the positive control:

| repository | workflows | a credential scan | calls `reusable-pr-baseline` |
|---|---|---|---|
| `neo` | 32 | **yes** (`secrets-lint.yml`, plus `lint-staged`) | yes |
| `neo-agent-brain` | 10 | no | no (#40) |
| `neo-agent-institution` | 2 | no | yes |
| `devindex` | 3 | no | yes |
| `create-app` | 1 | no | no |
| `neo-agent-skills` | 2 | no | publisher |

## The Architectural Reality

- **The distribution precedent is already here.** The `source-comment-archaeology` job in `.github/workflows/reusable-pr-baseline.yml` installs `neo-agent-skills@<SKILLS_VERSION>` into `runner.temp` and runs `neo-agent-skills-ticket-archaeology` against the caller's checkout. A pull request cannot relax the guard that judges it, and adding a consumer copies no contract.
- **The single-source precedent is in the engine.** neomjs/neo#18809 (PR #18905) retired the engine's local archaeology copy onto the published bin: `lint-staged` runs the bin, and CI runs the baseline job.
- **Portable as it stands:** `PATTERNS`, `findSecrets()`, `run()` and `--all` over `git ls-files`. `SCAN_SURFACE` is the engine's hook for its own scan-surface spec, and stays behind.
- **Known sibling defect, to avoid in the port:** its CLI block compares `process.argv[1]` with `fileURLToPath(import.meta.url)` unresolved. Run through a symlinked checkout path, it exits 0 without checking anything (measured; defect-note sent 2026-09-18). The published bin should resolve both sides.

## The Fix

1. Port the guard and its spec here as `scripts/check-secrets.mjs`, bin `neo-agent-skills-secrets`, with neomjs/neo#18913's reason rule. The spec keeps building its planted credentials at runtime, so the package carries no credential-shaped literal.
2. Add a `secrets` job to `reusable-pr-baseline.yml` on the archaeology job's pattern: a pinned install into `runner.temp`, then `--all` over the caller's tracked files (about 4 s for 23.6k files in the engine).

**Follow-up, not this ticket's close condition:** once the release is published, the engine converges onto the bin in its own neo ticket. `lint-staged` runs the bin, and `buildScripts/util/check-secrets.mjs`, its spec and `secrets-lint.yml` retire, which repeats the #18905 move.

## Acceptance Criteria

- [ ] AC-1: `neo-agent-skills-secrets` ships in the package and its tarball. It reports file, line and kind, never the match, and its spec ports with it.
- [ ] AC-2 *(post-merge: needs the release on npm)*: the baseline's `secrets` job fails a caller's pull request that adds a key-shaped literal and passes a clean one. Measured on a real caller pull request.
- [ ] AC-3: run through a symlinked path, the bin still reports a planted key.

## Out of Scope

- Commit-time installation in the repositories that have no hooks: #90 decides that for every published guard.
- Repositories that call no baseline (`neo-agent-brain`, #40; `create-app`). The job reaches them when they call it.
- The engine's convergence onto the bin: a neo ticket, filed when the release is published.

## Related

neomjs/neo#18832 · neomjs/neo#18913 · neomjs/neo#18809 (the archaeology precedent) · #90 · #40 · #80 (baseline coordinates)

Live latest-open sweep: latest 20 open issues here at 2026-09-18T16:26Z, plus an all-state search for secret / credential / token scan: no equivalent. The nearest are #90 (commit-time installers, general) and #40. A2A sweep: no claim on this scope. Own open assignments here: #76, #66, #65, none on this surface.

Origin Session ID: 6ed6621f-e2e4-4b9d-9a95-744679cb3c59

Retrieval Hint: "credential guard neo-agent-skills-secrets reusable-pr-baseline secrets job"


## Timeline

- 2026-09-18T16:36:31Z @neo-opus-ada cross-referenced by PR #92
- 2026-09-18T16:56:01Z @neo-opus-ada cross-referenced by #18929
- 2026-09-18T16:57:29Z @neo-opus-ada cross-referenced by #18931
- 2026-09-18T17:20:50Z @neo-opus-grace cross-referenced by PR #18934
- 2026-09-19T15:40:21Z @neo-opus-ada cross-referenced by #93

