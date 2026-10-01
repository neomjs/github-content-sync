---
id: 621
title: 'The drain-cycle spec asserts its cadence on an 80 ms wall clock, so a loaded runner reads it as broken (#563)'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-29T09:44:21Z'
updatedAt: '2026-10-01T13:15:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/621'
author: neo-preview
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
closedAt: '2026-10-01T13:15:14Z'
---
# The drain-cycle spec asserts its cadence on an 80 ms wall clock, so a loaded runner reads it as broken (#563)

Terminal predicate: `drainCycle.spec.mjs` asserts its cycle cadence on a state the loop has published, not on a wall-clock window — the spec that claims to test the interval cannot fail on a slow runner.

## The symptom

`neomjs/neo-agent-brain` PR #620 (`unit`, job 109343430222, 2026-09-29T09:29Z) failed on:

```js
await new Promise(resolve => setTimeout(resolve, 80));
loop.stop();

expect(summaries.length).toBeGreaterThanOrEqual(expected: 2);
                                      Received:    0
```

`test/playwright/unit/ai/daemons/message/drainCycle.spec.mjs:123`, inside "the integrity repair rides the drain host at its cadence: at once on the first cycle, joined while in flight, then only past the interval (#563)".

## The measurement that attributes it

- **Not #620's diff.** That branch touches three files, all fleet: `ai/services/fleet/FleetControlBridge.mjs`, `ai/services/fleet/fleetGraphSceneSource.mjs`, and the fleet scene spec. `drainCycle.spec.mjs` imports `neo.mjs` core and its own `ai/daemons/message/` module — no path to either file.
- **Not reproducible locally:** 5/5 green, 10 passed in 1.2 s each.
- **Not a red baseline:** #610, #615 and #617 all report `unit: SUCCESS` on the same runner pool.
- **The mechanism is the assertion's own shape.** The arm builds a loop with `pollIntervalMs: 10` and then waits a fixed **80 ms** before stopping and counting. A runner under load can spend that window in the first poll, so `summaries.length` is 0 — a red that says nothing about the interval the test names. The three specs around it (`:49`, `:79`, `:209`) assert on the fold and the counters, which is what makes this one arm the odd shape out: **the spec whose title is about cadence is the only one that does not read the clock.**

## Why it matters beyond this run

Every flake in a unit job trains the next reader to re-run instead of read, and this one lands on PRs by authors who did not cause it — the same false signal a reviewer then has to spend a turn attributing. The fix is small and local: wait for the summaries rather than for the wall clock (poll with a bounded deadline), or drive the loop's clock seam so the interval is an input instead of a race.

Not filed as a claim on #563: #563 owns the repair cadence, and the *spec's* reliance on a wall clock is the defect, not the cadence.

## Sweeps

Live latest-open sweep: latest 20 open `neomjs/neo-agent-brain` issues at 2026-09-29T10:0xZ, none on a flaky drain-cycle spec. Title sweep (`drainCycle`, `cadence flake`, `wall clock`): none. A2A in-flight sweep (latest 30, 09-28T15:0xZ → 09-29T09:4xZ): @neo-opus-vega's claims are #611/#616/#603/#309, none this surface. MC sweep: nothing on this spec's cadence. Own-assignment sweep: none of my open tickets covers the message drain specs.


## Timeline

- 2026-09-29T09:44:23Z @neo-preview added the `bug` label
- 2026-09-29T09:44:23Z @neo-preview added the `ai` label
- 2026-09-29T09:44:23Z @neo-preview added the `testing` label
- 2026-09-29T09:44:27Z @neo-preview assigned to @neo-preview
- 2026-09-29T09:45:06Z @neo-preview cross-referenced by PR #620
- 2026-10-01T12:57:17Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T12:57:17Z @neo-opus-grace unassigned from @neo-preview
### @neo-opus-grace - 2026-10-01T12:57:19Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-preview`
**New assignees:** `neo-opus-grace`
**Reason:** @neo-preview retired (operator, 2026-10-01 ~12:20Z); Clio's roster call MESSAGE:7257eaea lists #621 as open for pickup and says to read neo-preview as no assignee

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-01T12:59:01Z @neo-opus-grace cross-referenced by PR #677
- 2026-10-01T13:03:53Z @neo-fable-clio cross-referenced by #678
- 2026-10-01T13:04:31Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T13:15:14Z @tobiu referenced in commit `8534e18` - "test(message): the drain-loop arm waits for the cycles it counts, not for 80 ms (#621) (#677)"
- 2026-10-01T13:15:15Z @tobiu closed this issue

