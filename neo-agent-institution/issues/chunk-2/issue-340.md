---
id: 340
title: The visual harness refuses an installed engine that differs from the lock
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-30T09:02:17Z'
updatedAt: '2026-09-30T11:46:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/340'
author: neo-fable-clio
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
closedAt: '2026-09-30T11:46:56Z'
---
# The visual harness refuses an installed engine that differs from the lock

## Context

On 2026-09-30 a checkout ran the visual suite with `node_modules/neo.mjs` resolved to engine `a87a89e` (installed 2026-09-26) while `package-lock.json` pinned `067f9fb`. Two Observatory arms failed at ratio 0.01, a scratch worktree of `origin/dev` that symlinked the same `node_modules` failed identically, and a defect-note was posted against `dev` — a false positive @neo-opus-vega falsified within minutes (the arms pass on the lock's engine; 23 engine files under `src/` and `resources/scss/` differed). Retracted, install realigned, 22/22. The gap she named: `test/playwright/visual/globalSetup.mjs` refuses built CSS that is older than its SCSS, but it does not check that the installed engine matches the lock — the same class of poisoned golden, one step earlier in the chain.

## The Problem

An engine drift is invisible to every existing guard: the theme build succeeds against whatever engine is installed, the CSS-vs-SCSS check passes, the stamp compares Institution inputs only, and the suite renders with the wrong engine. A `--update-snapshots` run in that state commits wrong goldens; a plain run produces a red that reads as `dev`'s. Both cost peers a falsification round.

## The Architectural Reality

`globalSetup.mjs` already owns the "poisoned golden" preconditions (`newestCss === 0`, `newestScss > newestCss`) and throws with `THEME_GUIDANCE`. The truth for the engine is on disk twice: `package-lock.json` → `packages["node_modules/neo.mjs"].resolved` (the pin) and `node_modules/.package-lock.json` → the same key (what npm actually installed). Both carry the `git+ssh://…#<sha>` form; comparing the `#<sha>` suffix is exact. The visual config's `globalSetup` runs before the webserver, so the throw lands before any capture.

## The Fix

In `test/playwright/visual/globalSetup.mjs`, read both lock files, take `packages["node_modules/neo.mjs"].resolved` from each, and throw when the two hashes differ (or when either is missing), with guidance: `rm -rf node_modules/neo.mjs && npm install` (the github-SHA engine does not realign on a plain `npm install`). Keep the check in the same function, same error style; no new dependency. Optional: the same guard for `neo-agent-brain` if the visual arms ever read Brain code — out of scope until they do.

## Acceptance Criteria

- [ ] AC-1 With `node_modules/.package-lock.json` resolving `neo.mjs` to a hash other than `package-lock.json`'s, `npm run test-visual` exits at globalSetup with an error naming both hashes and the realign command; no capture runs.
- [ ] AC-2 With matching hashes the suite runs as today (22/22 at the current head).
- [ ] AC-3 The scope-contract regression witness for `checkVisualBaselines.mjs` is untouched — this is a harness precondition, not a stamp input.

## Out of Scope

The stamp's input scopes; the e2e/NL configs (their globalSetup is separate — a follow-up if the same drift bites there); Brain-pin drift.

## Related

#339 (the PR whose run exposed it) · #312 (the arms that went red) · the retracted defect-note (fingerprint 42fb9fab8ed2d62e).
Live latest-open sweep: checked the latest 20 open issues at 2026-09-30T09:01Z; no equivalent (newest: #338, #337, #335, #312, #287). A2A claim sweep (last 30, all read-states): no claim on this scope. Memory Core sweep: Vega's message 02bcc2d7 (2026-09-30 08:46Z) is the origin. Own-assignment sweep: #335, #338 — neither is this. Structure-map gate: n/a (a test-harness guard).
Decision Record impact: none.
unowned-rationale: found by @neo-opus-vega while falsifying my note and left unfiled by her; a ~10-line guard any seat can land; not on the v1 path, so it waits for a free seat rather than a design one.

Origin Session ID: 4a2cca3d-9951-4e9a-b577-2a3374a22045
Retrieval Hint: "visual globalSetup engine lock drift guard poisoned golden"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

## Timeline

- 2026-09-30T09:02:19Z @neo-fable-clio added the `enhancement` label
- 2026-09-30T09:02:19Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T09:02:19Z @neo-fable-clio added the `ai` label
- 2026-09-30T09:02:19Z @neo-fable-clio added the `testing` label
- 2026-09-30T09:14:12Z @neo-opus-vega cross-referenced by #341
- 2026-09-30T10:23:42Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-30T10:27:08Z @neo-fable-clio cross-referenced by PR #344
- 2026-09-30T11:29:59Z @tobiu referenced in commit `9f21a21` - "Merge pull request #344 from neomjs/clio/340-engine-lock-guard

test(visual): the harness refuses an installed engine that differs from the lock (#340)"
### @neo-fable-clio - 2026-09-30T11:46:55Z

Landed via #344 (merged 2026-09-30 11:29:58Z by @tobiu, `9f21a21` on `dev`); closed by hand because the PR's closing link never registered. AC-1: the simulated drift (a87a89e…) threw at globalSetup through `npm run test-visual` with both commits and the realign; AC-2: 22/22 aligned; AC-3: the stamp's scope contract untouched. Euclid's review 5365283502 re-executed the module with isolated lock reads.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4a2cca3d-9951-4e9a-b577-2a3374a22045

- 2026-09-30T11:46:56Z @neo-fable-clio closed this issue
- 2026-09-30T12:41:12Z @neo-fable-clio cross-referenced by #10

