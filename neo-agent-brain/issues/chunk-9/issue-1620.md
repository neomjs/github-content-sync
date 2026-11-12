---
id: 1620
title: KB ask specs share the synthesis rate cap across a worker's files
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T00:15:48Z'
updatedAt: '2026-11-12T01:05:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1620'
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
closedAt: '2026-11-12T01:05:40Z'
---
# KB ask specs share the synthesis rate cap across a worker's files

## Context

On Brain PR #1613 (head `7811cef3`), CI's `unit` job failed twice, a re-run of `suite (head)` included, on `test/playwright/unit/ai/services/knowledge-base/SearchService.spec.mjs:762` ("#1515 prioritizes query-relevant sections…"):

```
Error: expect(received).toContain(expected) // indexOf
Matcher error: received value must not be null nor undefined
Received has value: null
```

`capturedPrompt` is `null`, so `SearchService.ask()` never reached the stubbed model. The same CI run's base suite passes the test, the PR's previous head `280aa051` passed it, and it passes locally, including a full `CI=1` run with four workers. The diff touches no knowledge-base file.

## The Problem

`SearchService.ask()` gates synthesis on a per-minute cost cap: `checkAskRateLimit(this.askCallTimestamps, nowMs, aiConfig.askSynthesis.maxCallsPerMinute)` (`ai/services/knowledge-base/SearchService.mjs:865`). Over the cap it returns degraded references without calling the provider. The timestamps live on the singleton (`:197`), and a Playwright worker runs many spec files in one process, so every `ask()` that reached the model in that worker in the last minute counts.

The cap defaults to 20 (`NEO_KB_ASK_MAX_RPM`). `SearchService.spec.mjs` alone calls `ask()` 21 times, and `SearchService.reasoningEffort.spec.mjs` and `KnowledgeBase.TenantIsolation.spec.mjs` add more. None resets `askCallTimestamps`. With `workers: 4` and `fullyParallel: true` in CI, whether a test lands past the twentieth ask in its worker's minute depends on the file set and timing. Any PR can turn a later ask test red without touching the knowledge base.

## The Architectural Reality

- The cap is production behaviour (the runaway breaker), and its state on the singleton is right for a server process.
- In a test worker that state is shared fixture state. Each ask spec has to start below the cap, the same isolation rule as the stubs these specs already restore in `afterAll`.
- ADR-0019 rung (ii) allows binding `NEO_KB_ASK_MAX_RPM` in the harness environment before the first `ConfigProvider` import. That gives a deterministic red without timing.

## The Fix

Each spec file that calls `SearchService.ask()` resets `SearchService.askCallTimestamps = []` in `test.beforeEach`. Every test then starts below the cap whatever ran before it in the worker. The rate-cap unit tests, which call the pure helper `checkAskRateLimit`, are unaffected.

## Acceptance Criteria

- [ ] With `NEO_KB_ASK_MAX_RPM=5`, `SearchService.spec.mjs` fails on `dev` (the tests past the fifth model-reaching ask get no prompt) and passes with the fix.
- [ ] With the default cap, the four knowledge-base ask specs pass in one worker in either order.

## Out of Scope

- The cap itself, its default, and production behaviour.
- The unrelated one-off reds of `wake/daemon.spec.mjs` (defect-notes of 2026-11-11).

## Related

#1613 (blocked by this) · #1515 (the test) · `ai/mcp/server/knowledge-base/configBase.mjs:269` (the leaf)

Decision Record impact: none.

Live latest-open + closed sweep: the latest open Brain issues (red band, no owning ticket for KB spec isolation) plus a state-all search (`askCallTimestamps OR "rate cap" SearchService spec OR maxCallsPerMinute test`; no hits) at 2026-11-12T00:2xZ. A2A in-flight claim sweep: no claim on this surface. MC sweep: no prior record. Own-assignment sweep: #1607, #807, #1614, #1619, no overlap.

handoff: @neo-opus-grace self-assigns; it blocks #1613.

Origin Session ID: 15ea444b-707c-4458-bba5-79b4f61fb280

Retrieval Hint: "SearchService ask rate cap askCallTimestamps shared across spec files in a Playwright worker; capturedPrompt null flake"

## Timeline

- 2026-11-12T00:15:48Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-12T00:15:49Z @neo-opus-grace added the `bug` label
- 2026-11-12T00:15:50Z @neo-opus-grace added the `ai` label
- 2026-11-12T00:15:50Z @neo-opus-grace added the `testing` label
- 2026-11-12T00:17:48Z @neo-opus-grace referenced in commit `b2bb179` - "test(knowledge-base): the reasoning-effort spec cites ADR-0019 with the typed escape (#1620)"
- 2026-11-12T00:18:12Z @neo-opus-grace cross-referenced by PR #1621
### @neo-opus-grace - 2026-11-12T00:21:40Z

**Handover (Grace, session sunset 2026-11-12 ~00:25Z).** PR #1621 is open at `b2bb1797`; CI was 8 pass and 4 pending at handover. No reviewer requested yet: request one cross-family primary when CI is green.

Receipts: with `NEO_KB_ASK_MAX_RPM=5`, `SearchService.spec.mjs` fails 2 on `dev` and passes 30 with the fix; the four ask specs pass 56 in one worker at the default cap. After the merge, rebasing #1613 is the AC that proves it in CI. Owner: @neo-opus-grace.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 15ea444b-707c-4458-bba5-79b4f61fb280

- 2026-11-12T00:21:40Z @neo-opus-grace cross-referenced by #1607
- 2026-11-12T01:05:40Z @tobiu referenced in commit `7119bbf` - "test(knowledge-base): each ask spec starts below the synthesis rate cap its worker shares (#1620) (#1621)

* test(knowledge-base): each ask spec starts below the synthesis rate cap its worker shares (#1620)

* test(knowledge-base): the reasoning-effort spec cites ADR-0019 with the typed escape (#1620)"
- 2026-11-12T01:05:41Z @tobiu closed this issue

