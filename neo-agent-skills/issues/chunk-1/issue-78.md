---
id: 78
title: The baseline contract asserts an action version where it means a structure
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T17:39:54Z'
updatedAt: '2026-09-15T17:59:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/78'
author: neo-opus-vega
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
closedAt: '2026-09-15T17:59:38Z'
---
# The baseline contract asserts an action version where it means a structure

## Context

Found by the dependabot config from #74 on its **first** automated PR. `#77` bumped the actions group and `scripts/test-reusable-pr-baseline.mjs` went red. Diagnosed by @neo-opus-ada at @tobiu's request: the bump is correct and the test is wrong.

```
AssertionError: canonical reusable workflow violates its own contract
  actual:   [ 'missing caller checkout' ]
  expected: []
```

The contract has been version-coupled since it was written, and nothing surfaced it because nothing had bumped those actions. The first automated bump found it in seconds.

## The Problem

`scripts/test-reusable-pr-baseline.mjs:64`:

```js
['caller checkout', /uses: actions\/checkout@v4/, skillsJob],
```

The contract means *"the `skills-materialized` job checks out the caller's repository."* It expresses that as *"matches `checkout@**v4**`"*. Dependabot moved `v4 → v7`, the regex misses, and **a job that still has its checkout is reported as having lost it.**

A version pin standing in for a structural assertion, so every legitimate upgrade reads as the structure disappearing. Same defect class the org has hit repeatedly this week: a guard asking a **narrower** question than the condition it means to test, failing in the direction that looks like a real finding.

**Five further spots couple to a version** — one more than the initial report, which missed `:391`:

```
:265  mutation replaces the literal '- uses: actions/checkout@v4'
:316  fixture string embeds 'actions/setup-node@v4'
:317  fixture string embeds 'actions/setup-node@v4'
:391  mutation replaces a block containing 'actions/github-script@v7'
:407  mutation inserts 'actions/checkout@v4'
```

## The Architectural Reality

**These five are noisy, not dangerous, and the reason matters** — the initial report had it the other way round.

`expectMutationFailure` already carries the guard that makes a dead mutation impossible to miss:

```js
// scripts/test-reusable-pr-baseline.mjs:237
assert.notEqual(mutated, source, `${label}: fixture mutation changed nothing`);
```

So a `.replace()` whose target no longer exists does **not** pass for the wrong reason — line 237 fails loudly, naming the arm and saying the fixture mutation changed nothing. The recommendation to *"assert the replacement actually changed the source"* is already implemented. Whoever wrote that assertion anticipated exactly this, and it is why the next bump produces a clear failure instead of a silent hole.

That splits the work honestly: **`:64` is a correctness defect** (a false finding on a correct workflow), and the five `.replace()` sites are **upgrade friction** (every actions bump reds the suite until someone retypes a literal). Both worth fixing; only one is a false report.

## The Fix

**`:64` — assert the structure, not the release:**

```js
['caller checkout', /uses: actions\/checkout@v\d+/, skillsJob],
```

**The five `.replace()` sites — anchor on something version-free.** Prefer a target that carries the meaning (`- uses: actions/checkout@`, the `with:` block, the step `name:`) so a bump cannot invalidate it. Line 237 stays exactly as it is: it is the reason a mistake here is loud, and it should not be weakened into a warning.

## Decision Record impact

`none`. A test asserts what it already meant to assert.

## Acceptance Criteria

- [ ] `:64` matches any major — `checkout@v\d+` or equivalent — and the canonical workflow passes its own contract at the current action versions.
- [ ] `test-reusable-pr-baseline.mjs` exits 0 against `#77`'s bumped workflow (`checkout@v7`), which is the live reproducer.
- [ ] The five `.replace()` targets no longer embed a major version; each anchors on a version-free substring.
- [ ] `assert.notEqual(mutated, source, …)` at `:237` is **unchanged** — an AC rather than a note, because removing it while making the targets "safer" would trade a loud failure for a silent one.
- [ ] A regression arm proves `:64` accepts a bumped major **and still rejects a missing checkout** — the second half is what stops the fix from becoming a regex that matches anything.
- [ ] `node scripts/test-reusable-pr-baseline.mjs` and `node scripts/lint-skill-corpus.mjs --base origin/dev` both green.

## Out of Scope

- `#38` — how the baseline installs guards from the workflow's own commit. Adjacent, separately owned.
- Pinning actions by SHA instead of major. A policy change for `#14`, and `neomjs/neo`'s dependabot config deliberately tracks majors.
- `#77` itself: this ticket makes the contract accept the bump; merging the bump is that PR's business.

## Avoided Traps

⛔ **Do not "fix" the five `.replace()` sites by relaxing `:237`.** It is the control that makes a dead mutation loud. A version-agnostic anchor plus a strict `notEqual` is the safe combination; a loose anchor with a relaxed assertion is a suite that cannot fail.

**Do not widen `:64` to `checkout` alone.** `/uses: actions\/checkout/` would also match a commented-out line or a different action containing the word. `@v\d+` keeps it an action reference at a real version.

**Do not assume a red bump PR means a bad bump.** Here the bump was correct and the assertion was wrong — the automation's first act was to surface a latent coupling, which is the automation working.

## Related

- `#74` / `#75` — the dependabot config that surfaced this.
- `#77` — the live reproducer; this ticket unblocks it.
- `#14` — action-pinning policy, if SHA pinning is ever revisited.

**Sweeps.** Live latest-open: newest 12 open issues read at 2026-09-15T17:39:17Z (#76 … #40) plus an exact search on contract/checkout/version terms — `#38` (guard install source) and `#14` (governance) returned, neither is this. A2A in-flight claim sweep, latest 30 messages all read-states: @neo-opus-ada diagnosed `#77` and states she holds no seat and has not touched it; no competing claim. Memory Core rationale sweep on the problem's nouns: no prior decision on the contract's version coupling. Own-assignment sweep: `#51` is mine here and unrelated.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `reusable baseline contract checkout v4 version pin structural assertion mutation notEqual guard`

## Timeline

- 2026-09-15T17:43:50Z @neo-opus-vega cross-referenced by PR #79

