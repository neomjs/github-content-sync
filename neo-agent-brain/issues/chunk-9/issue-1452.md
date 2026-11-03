---
id: 1452
title: The KB core-sync spec builds two corpora under a 30 s unit timeout
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-11-03T02:29:11Z'
updatedAt: '2026-11-03T04:48:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1452'
author: neo-opus-vega
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
closedAt: '2026-11-03T04:48:53Z'
---
# The KB core-sync spec builds two corpora under a 30 s unit timeout

## Context

One unit arm keeps failing the Brain `unit` comparator on PRs that never touch the Knowledge Base: `DatabaseService.sync.spec.mjs:79`, "the exact Engine pin and Brain revision compile separately without consulting SourceRegistry".
- `dev` run 41502496432 (head suite): timed out at 30,000 and 30,004 ms, then passed on the third attempt in 27,853 ms.
- PR #1449's first run: timed out; a whole-run rerun went green.
- PR #1451, run 41563836418: timed out at 30,000 ms, and the comparator counted it as introduced.

Durations of the same arm when it passed, read from the uploaded unit reports:
- 15,800 ms (`dev` base, run 41502496432);
- 26,860 ms and 29,491 ms (#1450 head and base, run 41558337117).

## The Problem

The arm's real cost is 16–30 s, and it runs under Playwright's default 30 s test timeout, so a busy runner tips it over. The `unit` job diffs one run's head and base failure sets, so a timeout on the head side reads as introduced and blocks an unrelated PR. `gh run rerun --failed` reruns only the comparator, so every occurrence costs a whole-run rerun.

## The Architectural Reality

- `test/playwright/unit/ai/services/knowledge-base/DatabaseService.sync.spec.mjs:79` runs `DatabaseService.createKnowledgeBase` for real. It compiles the pinned Engine corpus and the Brain corpus to JSONL, then asserts both manifests: the revisions, and real module parents such as `examples/component/timer/MainContainer.mjs`. Those assertions need the real corpora, so a fixture would hollow the arm out.
- `test/playwright/playwright.config.unit.mjs` sets no `timeout`; CI adds `retries: 2`. Its own note (around line 88) says a loose timeout is no wall-clock *budget*, because it survives contention. This arm tests no timing, so a timeout that survives contention is exactly what it needs.
- Per-test budgets are an established idiom: `test.setTimeout(…)` in `Orchestrator.spec.mjs`, `daemon.spec.mjs` and `daemonDeliveryOwner.spec.mjs`.
- Design authority: n/a. This is a test budget, not designed behavior; the failing runs are the evidence.

## The Fix

Give this one arm its own budget, `test.setTimeout(120_000)`, with a one-line basis: the slowest observed pass is 29.5 s, so 4× that survives a busy runner. The config default stays as it is.

## Acceptance Criteria

- [ ] AC-1: The arm sets its own timeout with its measured basis in one line; the unit config's default is unchanged.
- [ ] AC-2: The fix PR's `unit` job passes. A timeout can still show up in the base suite, which runs old code, but the comparator then reads it as fixed, not introduced.

## Out of Scope

- Making `createKnowledgeBase` faster.
- Teaching the comparator to retry a head-only failure before calling it introduced. That is the general cure for any flake, and a separate change.

## Avoided Traps

- **Raising the config-wide default:** that would loosen every unit spec to cover one corpus build.
- **Moving the arm to the integration suite:** it would change CI topology for a one-line problem, and the arm guards the core-sync contract in the unit suite.

## Related

- #282: the spec's origin.
- #201: the retained Brain unit suite.
- #321: the defect ledger's machine producer for CI failures.

*Sweep attestations:*
- *Live latest-open + closed sweep:* the latest 20 open, plus a closed search ("timeout flake unit spec"), at 2026-11-03T02:3xZ. Hits: #1032, #321 and #89, all with a different surface. No equivalent.
- *A2A in-flight sweep:* the last 30 messages, all statuses. No claim.
- *Memory Core sweep:* `query_raw_memories` on the symptom nouns. No prior decision.
- *Own-assignment sweep:* no overlap.
- *Structure map (§1c):* N/A, a timeout inside an existing spec with no placement change.

Ownership: self-assigned (Vega); the fix follows immediately, because the flake blocks every Brain PR it lands on.

Origin Session ID: afcce2e4-5eb9-4fa6-8895-4730263a042d

Retrieval Hint: "DatabaseService.sync spec timeout core corpus createKnowledgeBase unit comparator flake introduced"

## Timeline

- 2026-11-03T02:29:12Z @neo-opus-vega added the `bug` label
- 2026-11-03T02:29:13Z @neo-opus-vega added the `ai` label
- 2026-11-03T02:29:13Z @neo-opus-vega added the `testing` label
- 2026-11-03T02:29:18Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-03T02:29:52Z @neo-opus-vega cross-referenced by PR #1453
### @neo-opus-vega - 2026-11-03T03:05:19Z

**The comparator question, with tonight's evidence: a fork on its record, not a fix.**

PR #1451's head (`45c6ec5b`) failed `unit` on three consecutive whole-run attempts, each time on a different spec its diff cannot reach. Every spec passed at the same head on another attempt:
1. `DatabaseService.sync.spec.mjs:79`, the timeout this ticket fixes;
2. `TenantRepoSyncService.spec.mjs:7297`, a live lease owner reclaimed under load;
3. `FleetBridgeKeyProvisioning.spec.mjs:427`, the stale-listener replacement arm.

`compareUnitFailures.mjs`'s record makes this behavior intentional. A red that depends on the host is assumed to "show on both sides and cancel". A pass that needed a retry counts as failing (#650), and every introduced test reaches the defect ledger's CI ingest. Blocking is the forcing function, and the intended remedy is fixing each flaky spec, which is what this ticket does for (1).

The part tonight falsifies is the cancel premise. A load-dependent red lands on one side only, so it never cancels and blocks unrelated PRs. Whether that cost stays acceptable, or a head-only failure should be re-run in isolation before it counts as introduced, is a decision for the comparator's record (#650's line), not a fix filed against it. Until then, (2) and (3) reach the ledger through its ingest and take the same per-spec route as (1).

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-03T04:48:53Z @tobiu referenced in commit `15dbfb3` - "test(knowledge-base): give the core-sync arm a budget for its two corpus builds (#1452) (#1453)

The arm compiles the pinned Engine corpus and the Brain corpus for real, 16–30 s on CI runners, under Playwright's 30 s default, so a busy runner failed it and the unit comparator blocked unrelated PRs as introduced. It now sets its own 120 s timeout; the unit config's default is unchanged."
- 2026-11-03T04:48:54Z @tobiu closed this issue

