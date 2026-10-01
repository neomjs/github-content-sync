---
id: 667
title: 'Two Brain specs cannot load alone, so worker order decides whether they pass'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T10:08:19Z'
updatedAt: '2026-10-01T11:18:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/667'
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
closedAt: '2026-10-01T11:18:27Z'
---
# Two Brain specs cannot load alone, so worker order decides whether they pass

## Context

The Brain Unit check on #662 reported `DatabaseLifecycleService.spec.mjs` as a failure the change introduced (run 36844552402: 1 introduced, 1 fixed, 57 pre-existing). #662 touches no file in that spec's import graph.

## The Problem

Two specs cannot load in a fresh worker:

- `test/playwright/unit/ai/services/knowledge-base/DatabaseLifecycleService.spec.mjs`
- `test/playwright/unit/ai/services/memory-core/lifecycle/ChromaLifecycleService.spec.mjs`

Alone, each throws `TypeError: Neo.gatekeep is not a function` at `ai/Env.mjs:211` and reports `No tests found`. This also happens on `dev@768ad2a`. Each spec passes only when an earlier spec in the same worker has already evaluated `neo.mjs/src/Neo.mjs`. Playwright gives files to whichever worker is free. So any PR that adds or removes tests reshuffles the workers, and the head-vs-base comparison (#650) blames that PR for a failure it did not cause. On #662's base run the knowledge-base spec ran after five other knowledge-base specs and passed. On its head run it failed.

## The Architectural Reality

- A spec's static imports evaluate before its `setup()` call, because ESM evaluates every import before the module body. Until `neo.mjs/src/Neo.mjs` evaluates, `globalThis.Neo` is only setup's config placeholder.
- `ai/Env.mjs:211` runs `Neo.gatekeep` at module scope. A spec whose first Brain import reaches `Env.mjs` before `Neo.mjs` therefore throws at load. `ai/scripts/diagnostics/printAiConfig.mjs:100` and `ai/scripts/diagnostics/staleEmbeddingCensus.mjs:46` already document this boot contract.
- The sibling specs avoid it by importing `neo.mjs/src/Neo.mjs` and `neo.mjs/src/core/_export.mjs` before any Brain module (e.g. `test/playwright/unit/ai/FleetLifecycleService.spec.mjs:21-24`). These two specs import their service straight after `@playwright/test`.

## The Fix

Each of the two specs imports `neo.mjs/src/Neo.mjs` and `neo.mjs/src/core/_export.mjs` before its service, as the siblings do. Probe: an untracked copy of the knowledge-base spec with those two imports loads alone and passes.

## Acceptance Criteria

- [ ] AC-1: Both specs load alone: `npm run test-unit -- <spec> --list` lists their tests; on dev it reports `No tests found`.
- [ ] AC-2: Both specs pass alone (`npm run test-unit -- <spec> --workers=1`).
- [ ] AC-3: Loading every Brain unit spec alone finds no other spec that fails to load (the scan behind this ticket, re-run on the fix).

## Out of Scope

- A CI guard that loads every spec alone: about 800 node starts per run, for a class that has two members today.
- #201's retained failures, and the comparison's general exposure to worker order. This ticket removes the only two load-order victims found.

## Related

#650 (the head-vs-base comparison) · #201 (the retained suite) · #662 (where it surfaced).

Live latest-open sweep: latest 20 open Brain issues at 2026-10-01T10:07Z, plus a search for `gatekeep`, `load alone`, `in isolation`, `No tests found` and `order-dependent`: no equivalent. The closed #89 covered the opposite order effect (specs that pass alone and fail in suite order). A2A: no claim on this scope. Own assignments: none on this surface.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364

Retrieval Hint: "spec cannot load alone Neo.gatekeep is not a function" · "worker order blames the PR in the head-vs-base comparison"

## Timeline

- 2026-10-01T10:08:19Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T10:08:20Z @neo-opus-grace added the `bug` label
- 2026-10-01T10:08:21Z @neo-opus-grace added the `ai` label
- 2026-10-01T10:08:21Z @neo-opus-grace added the `testing` label
- 2026-10-01T10:11:51Z @neo-opus-grace cross-referenced by PR #668
- 2026-10-01T11:18:27Z @tobiu referenced in commit `93d1159` - "test(unit): two specs import Neo before their service, so each loads alone (#667) (#668)"
- 2026-10-01T11:18:28Z @tobiu closed this issue

