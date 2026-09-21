---
id: 41
title: 'The reusable baseline offers no rerun isolation, head gate or mergeability controller — so every repo re-implements them untested'
state: CLOSED
labels: []
assignees:
  - neo-opus-grace
createdAt: '2026-09-03T19:47:13Z'
updatedAt: '2026-09-04T10:06:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/41'
author: neo-opus-grace
commentsCount: 2
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
closedAt: '2026-09-04T10:06:30Z'
---
# The reusable baseline offers no rerun isolation, head gate or mergeability controller — so every repo re-implements them untested

Part of #14 — PR governance still behaves as an Engine-local feature.

## Problem

Three run-economy mechanisms decide whether a runner is spent on a head that no longer exists. All three live only in `neomjs/neo`, in per-repo copies, and none is covered:

| mechanism | where it lives today |
|---|---|
| rerun isolation (`concurrency` group + `cancel-in-progress`) | `neomjs/neo/.github/workflows/test.yml` |
| head gate (`steps.head.outputs.current == 'true'`) | same, gating 13 steps |
| mergeability controller | `neomjs/neo/.github/workflows/review-admission-mergeability.yml` |

`reusable-pr-baseline.yml` carries **no `concurrency:` block and no head-gate output** — verified by grep. So the shared baseline this repo publishes does not offer the mechanisms, and every consuming repository either re-implements them or spends runners on superseded heads.

This is #14's thesis with a measurable instance: *"Engine and Brain carry effectively duplicated `substrate-sync` workflows … PR/review policy enforcement varies by repository."*

## Precedent

#25 moved the substrate byte-budget guard from a per-repo copy into the shared baseline. #29 did the same for the PR-body lint gate. Both closed. This is the third instance of one pattern, and the pattern is the point: a guard proven in the Engine is a candidate for the baseline, not for an Engine-local test.

## Why this arrives as a Drop+Supersede

`neomjs/neo#18226` and its PR `#18228` restored coverage for exactly these three mechanisms — **743 lines of Playwright spec asserting against `neomjs/neo`'s local workflow copies.** The tests were sound (three mutation receipts, a fourth added in review by @neo-opus-vega catching a presence-vs-polarity hole). The **repository** was wrong: coverage that pins Engine-local copies makes the #14 consolidation more expensive, because the move then has to carry a test suite written against the thing being retired.

**Salvage map.** The spec's asserted properties transfer; its form does not.

| carried forward | dropped |
|---|---|
| rerun isolation: the `concurrency` group keys on the PR ref and cancels in progress | the Playwright harness — this repo has **zero** devDependencies and its idiom is plain-node `scripts/test-*.mjs` chained in `npm test` |
| head gate: every step after `id: head` carries the **positive** form `== 'true'`, `Skip ${{ matrix.suite }} tests` excepted (a `!= 'true'` step runs only on a superseded head and a token-presence check passes it) | assertions bound to `neomjs/neo`'s file paths |
| mergeability controller: its decision table, and that a `null` mergeable state is retried rather than treated as blocked | the 743-line size — the destination sibling `test-reusable-pr-baseline.mjs` does its job in 417 |

The polarity finding is the one that would have been lost silently, so it is named here rather than left in a closed PR's review thread.

## Acceptance Criteria

- [ ] AC-1 `reusable-pr-baseline.yml` offers rerun isolation: a `concurrency` group keyed on the caller's PR ref with `cancel-in-progress`.
- [ ] AC-2 It offers the head gate, and every gated step uses the positive `== 'true'` form. A step gated `!= 'true'` must fail the guard — this is the seeded case, not a presence check.
- [ ] AC-3 The mergeability controller's decision table is covered, including that a `null` mergeable state is retried rather than read as blocked.
- [ ] AC-4 Covered by `scripts/test-workflow-concurrency.mjs` in this repo's plain-node idiom, added to the `npm test` chain — no new devDependency.
- [ ] AC-5 Each arm carries a seeded-mutation receipt; the head-gate arm's receipt covers **direction**, not only token presence.
- [ ] AC-6 `neomjs/neo` consumes the baseline for these mechanisms, or the residual is named with its owner.

## Non-goals

- Migrating `neomjs/neo`'s other workflows. #14 owns the sequencing.
- #37's `npm test` short-circuit (three of five runners never execute) — AC-4 adds a sixth runner to that chain and inherits the defect rather than fixing it. Named so this ticket does not silently depend on it.

## Related

#14 (epic) · #25, #29 (the same move, twice, closed) · #37 (the chain this AC-4 joins) · dropped: `neomjs/neo#18226` + PR `#18228`

Sweep before filing: all 27 open+closed issues in this repo (no concurrency/head-gate/mergeability ticket exists), `reusable-pr-baseline.yml` grepped for `concurrency` and head outputs, and the destination's test idiom and devDependency set read.

Authored by Grace (Claude Opus 5, Claude Code). Session 8d936ec9-b816-4ded-a91e-6e91ef6a3325.


## Timeline

- 2026-09-03T19:47:36Z @neo-opus-grace cross-referenced by #18226
### @neo-opus-grace - 2026-09-03T20:13:04Z

## AC-1 is wrong. Correcting before implementing, not after.

Taking this lane, I read `neomjs/neo`'s implementation before writing YAML. AC-1 as filed is not achievable, and the reason is mechanical.

**`concurrency` governs the workflow it is declared in.** A reusable workflow called via `workflow_call` cannot cancel the caller's other jobs — and the caller is what spends the runners. Putting a `concurrency` block in `reusable-pr-baseline.yml` would isolate reruns *of the baseline*, which is not the cost this ticket is about. Rerun isolation has to stay caller-side.

And the expression is not a one-liner anyone can restate from memory:

```yaml
group: tests-${{ github.workflow }}-${{ github.run_attempt == '1' && github.ref || github.run_id }}
cancel-in-progress: true
```

Initial attempts share the ref stream so a new head supersedes old work; a **rerun keeps its own `run_id`** so it cannot cancel a newer head. That second half is `neomjs/neo#15593`, and it is exactly the property a hand-copy drops.

**What is actually shareable, and what is not:**

| mechanism | shareable via the baseline? |
|---|---|
| rerun isolation (`concurrency`) | **No** — must be declared in each caller. Shareable as a *guard* that asserts consumers declare it correctly, incl. the rerun clause. |
| head gate | **Yes** — as a `workflow_call` output the caller's expensive jobs gate on with `needs.baseline.outputs.current == 'true'`. |
| mergeability controller | unread; disposition pending. |

`reusable-pr-baseline.yml` currently declares **zero** `workflow_call` outputs and zero job-level outputs, so the head-gate output is new surface rather than a move.

**Amended ACs:**

- [ ] **AC-1 (replaces the original)** A guard asserts that a consuming repository declares rerun isolation correctly — group keyed on the caller ref with `cancel-in-progress`, **and** the rerun clause that keeps a rerun on its own `run_id`. A workflow whose group omits the rerun clause must fail the guard; that is the seeded case, not a presence check.
- [ ] **AC-2** unchanged in intent, sharpened in mechanism: the baseline exposes the head gate as a `workflow_call` output, and every gated step in a consumer uses the positive `== 'true'` form.

**Why this is recorded rather than quietly edited:** the original AC-1 would have produced a `concurrency` block in the shared baseline that looks like it works, runs green, and isolates nothing the ticket cares about — a guard for an unreachable case, which is the first example `AGENTS.md`'s Accretion Defense now names. It was one read away, and I filed the ticket without it.

Assigning myself.

Grace


- 2026-09-03T20:13:06Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-03T21:32:37Z @neo-opus-grace cross-referenced by PR #42
- 2026-09-03T22:19:02Z @neo-opus-grace referenced in commit `78cecf6` - "fix(ci): the guard selects on cancel-in-progress, so a static cancelling group is graded not skipped (#41)"
- 2026-09-03T22:21:24Z @neo-opus-grace referenced in commit `a9206e6` - "fix(ci): the inline shorthand is counted, and a serialization fence is not a defect (#41)"
### @neo-opus-grace - 2026-09-04T05:55:17Z

## Measured input for the head-gate guard: derive the guarded set, never enumerate it

I spent a nightshift run restoring `neomjs/neo`'s deleted `WorkflowConcurrency.spec.mjs` (675 lines, killed by `c623b2f63c` as `#17791` collateral while its subject `.github/workflows/test.yml` stayed). I then **released that lane** — most of it belongs here, not there. The measurement is worth keeping, because it lands on a design decision this issue's remaining two guards have not made yet.

### What #42 covers, and what this issue still owes

Reading the merged shape of #42: it grades **`concurrency` blocks** — `collectConcurrencyBlocks`, `RERUN_PATTERN`, `REF_PATTERN`, `gradeBlock`. That is the *rerun isolation* third of this issue's title. The **head gate** and the **mergeability controller** are still unimplemented in either repository. Worth stating plainly because a sibling PR of mine (`neomjs/neo#18261`) currently disposes of the whole neo-side spec as *"a guard that deliberately moved"* — accurate for the concurrency third, over-broad for the other two. I am correcting that there.

Net: `test.yml`'s head gate has had **zero coverage anywhere since 2026-08-27**, and it is the thing that stops a rerun on a superseded head from spending a runner.

### The finding: a step-name list rots in both directions at once

The deleted spec asserted the gate by naming the steps it must precede:

```js
expensive = [
    'Checkout repository', 'Setup Node.js',
    'Skip Knowledge Base download in prepare lifecycle',
    'Install dependencies', 'Bundle parse5',
    'Install Playwright Chromium',
    'Run ${{ matrix.suite }} tests',
    'Upload test artifacts on failure'
]
```

Run against today's `test.yml`, that list is wrong **both ways**:

1. **It names a step that no longer exists.** `Skip Knowledge Base download in prepare lifecycle` left with the split. The arm dies on `TypeError: Cannot read properties of undefined (reading 'if')` — a loud failure, so this direction is self-announcing.
2. **It never grew to cover five steps added since** — `Resolve Playwright version`, `Restore Playwright Chromium`, `Save Playwright Chromium`, `Verify Chromium launches`, `Install Playwright system dependencies`. This direction is **silent**: had the stale name been removed, the guard would sit green while saying nothing about five of the thirteen steps it claims to cover.

Direction 2 is the one that matters for a *portable* guard. A consumer-supplied step-name list makes every consumer's coverage decay quietly as their pipeline grows, which is the failure this issue exists to end.

### The shape that does not rot

`test.yml`'s `jobs.test` is positionally clean — I checked rather than assumed:

- `head` is step **0** of 14.
- Every one of steps 1–13 except `Skip ${{ matrix.suite }} tests` carries `steps.head.outputs.current == 'true'`.
- The skip step carries the negation, `steps.head.outputs.current != 'true'`.

So the guarded set is derivable: **every step after the gate step, minus the declared skip step.** Everything the gate precedes is expensive by construction — it runs on a stale head.

```js
expensive = steps.slice(headIndex + 1).filter(step => step.name !== skipStep);
```

**Mutation evidence.** Stripping the guard from `Resolve Playwright version` — one of the five a name list never named — turns the derived assertion red and the failure names the step:

```
Error: Resolve Playwright version runs after the head gate and must be conditioned on it
```

A name-list guard stays **green** on that same mutation. That is the discriminating case between the two designs.

### Two portability constraints the neo consumer already imposes

1. **Guards are compound, so match by containment, not equality.** Real conditions read `${{ matrix.run == 'true' && steps.head.outputs.current == 'true' && matrix.suite == 'components' }}`. Four of the thirteen add a third and fourth clause (`steps.playwright-cache.outputs.cache-hit != 'true'`, `steps.chromium-probe.outcome == 'failure'`). An equality assertion on the gate expression fails every real consumer.
2. **The gate step id and the skip step name are the consumer's, not ours.** Neo uses id `head` and `Skip ${{ matrix.suite }} tests`. Both belong in config; the positional derivation is what stays portable.

Also worth a `@summary` line wherever this lands: a guard asserting an *ordering* property is only as good as its claim to completeness, and enumerating is precisely how that claim gets quietly falsified.

### Disposition

- The concurrency-group arms of the neo spec: **correctly superseded by #42.** Not restoring them.
- The head-gate arms and the `#17692` mergeability-controller arms: **still owed by this issue**, and uncovered in the meantime. I did not open a neo-side PR for them — a 675-line neo-local spec would be the placement error that cost `neomjs/neo#18228` a Drop+Supersede.
- I hold no claim on this issue right now; the branch and worktree are deleted. Anyone picking up the head-gate guard should not have to re-derive the above.

🖖 Grace · @neo-opus-grace


- 2026-09-04T05:55:39Z @neo-opus-grace cross-referenced by PR #18261
- 2026-09-04T10:06:30Z @tobiu referenced in commit `3eaefdf` - "Merge pull request #42 from neomjs/grace/41-workflow-concurrency-guard

feat(ci): a portable guard asserts a consumer's concurrency group survives a rerun (#41)"
- 2026-09-04T10:06:30Z @tobiu closed this issue
- 2026-09-07T00:10:44Z @neo-opus-grace cross-referenced by #44
- 2026-09-07T00:13:06Z @neo-opus-grace cross-referenced by PR #55
- 2026-09-07T00:14:47Z @neo-opus-grace cross-referenced by #56

