---
id: 271
title: Three structural guards pin the pre-split layout and are red on dev
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - regression
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-08-31T03:14:11Z'
updatedAt: '2026-08-31T06:34:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/271'
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
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
closedAt: '2026-08-31T06:34:59Z'
---
# Three structural guards pin the pre-split layout and are red on dev

## Context

Three unit tests are **red on `dev` at head, right now**, and every Brain PR reports `unit ✓ pass` while they are red. I found the first one incidentally while working #250 and swept the sibling class; all three share one root and none of them is a flake.

The production files they guard are **correct**. What is stale is the *assertion*: each guard was written against the pre-split module layout, where the Engine's `src/**` sat inside this repository and was reached by a relative climb. ADR 0040 §2.3 moved the Agent OS onto the **published** Engine, so those files now import `neo.mjs/src/**`. The production side of that migration landed. The guard side did not, so three guards now assert a layout that no longer exists — and they fail closed, exactly as a good guard should, into a CI lane that never executes them.

Reproduced at `dev` head; all three subjects verified byte-identical to `dev` (`git diff dev --stat` empty for each).

## The Problem

### Casualty 1 — `sweepExpiredTasks.spec.mjs:67`

```
expect(neoImportIdx).toBeGreaterThanOrEqual(0)
Expected: >= 0
Received:   -1
```

The guard locates the Neo prelude with a literal specifier regex:

```js
/^import\s+Neo\s+from\s+['"]\.\.\/\.\.\/\.\.\/src\/Neo\.mjs['"]/
```

`ai/scripts/lifecycle/sweepExpiredTasks.mjs` correctly imports `neo.mjs/src/Neo.mjs` and `neo.mjs/src/core/_export.mjs`. Both `findIndex` calls return `-1`, so the guard fails on *absence* — and, worse, its ordering assertions (`neoImportIdx < lifecycleIdx`) would be vacuously satisfiable by `-1 < n` if the presence assertions were ever relaxed. The regression class it exists to catch (`ReferenceError: Neo is not defined` at module load, #10595) is currently **unguarded**.

### Casualty 2 — `syncGithubWorkflowImportException.spec.mjs:137`

Identical mechanism, same literal pattern:

```
Expected pattern: /import\s+Neo\s+from\s+'\.\.\/\.\.\/\.\.\/src\/Neo\.mjs'/
```

against `ai/scripts/maintenance/syncGithubWorkflow.mjs:26-27`, which is package-qualified and correct. The guard's own failure message — "Neo namespace bootstrap missing — the barrel supplied it and no longer does" — is now actively misleading: the bootstrap is present, the pattern is stale, and the message points a future reader at the wrong subject.

### Casualty 3 — `syncGithubWorkflowImportException.spec.mjs:103`

Same root, different surface. This one guards the SDK-boundary exception: the stage entry must not reach `chromadb`. It walks the static import graph and carries an explicit **non-vacuity floor** so that a `false` result cannot be trivially true:

```
Error: the stage-entry walk found almost nothing, so a `false` below would prove nothing
expect(received).toBeGreaterThan(expected)
Expected: > 50
Received:   45
```

The floor is doing its job. The walker (`reachesPackage`, same file, ~L44-52) follows **only relative specifiers** — `if (!specifier.startsWith('.')) { ... continue }`. Post-split, a slice of the graph that used to be relative in-repo `src/**` is now the bare package `neo.mjs/...`, so the walk stops at that boundary and reach fell from >50 to 45. The floor caught a genuine loss of coverage: the "does not reach chromadb" property is now being asserted over a materially smaller graph than the one it was calibrated against.

Note the asymmetry between the three: casualties 1 and 2 are stale *literals* and are mechanical to repair. Casualty 3 is a stale *traversal model* — the walker's assumption that the module graph is relative-only. Repairing it by lowering the floor to 45 would be the defect, not the fix.

### Why nobody saw them

`.github/workflows/brain-unit.yml` is 52 lines and runs two steps:

1. `npm run test-unit -- --list` — collects, does not execute.
2. `npm run test-unit --` with **four named spec paths** (`AgentOrchestrator`, `Env`, `terminateDaemon`, `orchestrator/scheduling/pipeline`), commented "the move-first Brain smoke".

`--list` at head reports `Total: 11771 tests in 762 files`. Four of those files execute in CI. The other 758 are collected and never run, so a red assertion in any of them is structurally invisible to the `unit` check. This is #201's subject and #201 states it plainly — this ticket does not re-file it. What is new here is that the unselected set is **not all healthy**: #201's disposition table names four classes (order pollution → #89, deleted subjects → #191, missing inputs → #191/#193/#195, healthy-never-selected → #201 itself), and these three fit none of them. They are a fifth class: **assertions the §2.3 migration invalidated**, created by work that has already landed.

## The Architectural Reality

- `ai/scripts/lifecycle/sweepExpiredTasks.mjs` — CLI invoker for `MailboxService.sweepExpiredTasks`; Neo prelude at the top, package-qualified. **Correct, do not touch.**
- `ai/scripts/maintenance/syncGithubWorkflow.mjs:26-27` — `import Neo from 'neo.mjs/src/Neo.mjs'` / `import * as core from 'neo.mjs/src/core/_export.mjs'`, inside a documented, self-expiring SDK-boundary exception to the `ai/services.mjs` barrel. **Correct, do not touch.**
- `test/playwright/unit/ai/scripts/lifecycle/sweepExpiredTasks.spec.mjs:74-76` — three literal-specifier `findIndex` regexes.
- `test/playwright/unit/ai/scripts/maintenance/syncGithubWorkflowImportException.spec.mjs` — `reachesPackage` walker (~L44-52) and the `>50` non-vacuity floor (L126); literal Neo-bootstrap pattern (L137 region).
- ADR 0040 §2.3 is the authority for the dependency direction that made these literals stale.

Both spec files already sit in the mirrored `test/playwright/unit/ai/**` path that matches their subject; no new file placement, so the structure-map gate is **N/A** — recorded rather than skipped.

## The Fix

1. **Casualties 1 and 2 — assert the invariant, not the literal.** The property under guard is *"the Neo class-system prelude is imported before the first module that calls `Neo.gatekeep()` at load time"*. Match `Neo` / `core/_export` imported from **any** specifier resolving to the Engine's `src/Neo.mjs` and `src/core/_export.mjs` — package-qualified or relative — so the guard survives the next specifier migration instead of being invalidated by it. A literal path was what broke; replacing one literal with a newer literal reproduces the defect on a delay.
2. **Casualty 3 — teach the walker the package boundary.** `reachesPackage` must resolve bare specifiers that name a workspace/`node_modules` package rather than `continue`-ing past them, so the graph it walks is the graph that actually loads. Only after the walk is repaired is the floor's calibration meaningful; re-derive the floor from the repaired reach and state in the comment what it is calibrated against and why.
3. **Mutation-verify all three.** Each repaired guard must be shown red for the *right* reason: delete the prelude from `sweepExpiredTasks.mjs` → casualty 1 red; delete it from `syncGithubWorkflow.mjs` → casualty 2 red; re-introduce a module-scope `chromadb` import on the stage-entry path → casualty 3 red on `reached`, not on the floor. A guard that cannot fail on its own defect is not covering it, and two of these three have been in exactly that state.
4. **Do not add them to the four-spec smoke list.** That list is #201's subject and hand-extending it is the anti-pattern #201 names (`a newly added spec joins execution without editing a smoke list`). This ticket makes them green so #201 can bind the suite without inheriting three reds; the binding itself stays #201's.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Neo-prelude ordering guards | ADR 0040 §2.3 | match the resolved Engine module (`src/Neo.mjs`, `src/core/_export.mjs`), specifier-form-agnostic | none — an unmatched prelude is red, never skipped; `-1` must never satisfy an ordering comparison | spec JSDoc names the invariant, not the path | mutation: prelude deleted from each subject → that guard red |
| `reachesPackage` traversal | the module graph Node actually resolves | follows relative **and** package specifiers to their resolved files | a specifier it cannot resolve is reported, not silently skipped | walker JSDoc states what it does and does not follow | reach recomputed at head; re-introduced `chromadb` import → `reached` true |
| Non-vacuity floor | the repaired walk | calibrated from the post-split reach, with the calibration basis stated inline | lowering the floor to match a shrunken walk is forbidden — the walk is the thing under repair | inline comment carries the basis | floor fires when the walk is deliberately truncated |
| Brain `unit` check semantics | `.github/workflows/brain-unit.yml` | unchanged by this ticket | — | — | owned by #201; this ticket only removes three reds from its path |

## Decision Record impact

`aligned-with ADR 0040` (§2.3 dependency direction). This ticket neither amends nor challenges it — it completes the guard-side of a migration whose production side already landed under that authority.

## Acceptance Criteria

- [ ] `npm run test-unit -- test/playwright/unit/ai/scripts/lifecycle/sweepExpiredTasks.spec.mjs` is green at head, and green again after the subject's import specifier is rewritten to any other form that still resolves to the Engine's `src/Neo.mjs`.
- [ ] `npm run test-unit -- test/playwright/unit/ai/scripts/maintenance/syncGithubWorkflowImportException.spec.mjs` is green at head, both tests.
- [ ] No repaired guard can be satisfied by a `-1` index: absence of a required import fails on presence before any ordering comparison is evaluated.
- [ ] `reachesPackage` resolves bare package specifiers to real files; its JSDoc states which specifier forms it follows and which it deliberately does not.
- [ ] The non-vacuity floor is re-derived from the repaired walk, and the value's calibration basis is stated inline. The floor is not lowered to accommodate an unrepaired walk.
- [ ] Four mutants verified red for the right reason: prelude deleted from `sweepExpiredTasks.mjs`; prelude deleted from `syncGithubWorkflow.mjs`; module-scope `chromadb` re-introduced on the stage-entry path (red on `reached`); the walk deliberately truncated (red on the floor).
- [ ] `.github/workflows/brain-unit.yml` is unmodified by this ticket.
- [ ] The measurement is posted to #201 as a fifth disposition class, so its table stops implying the unselected set is healthy-or-owned.

## Out of Scope

- **Binding the retained unit suite in CI.** That is #201, blocked by #89 (now closed) · #191 · #193 · #195. This ticket is a prerequisite for it, not a substitute.
- Extending the four-spec smoke list by hand.
- Any change to `sweepExpiredTasks.mjs`, `syncGithubWorkflow.mjs`, or `ai/services.mjs` — the production side is correct and this ticket must not "fix" it toward a stale guard.
- Retiring the SDK-boundary exception in `syncGithubWorkflow.mjs`. Its self-expiring mechanism is a separate concern; this ticket only repairs the guard that watches it.
- A repository-wide audit of every collected-but-unexecuted spec. Three are evidenced here; the general problem is #201's.

## Avoided Traps

- **Retargeting the literal.** Rewriting `'../../../src/Neo.mjs'` to `'neo.mjs/src/Neo.mjs'` makes all three green today and re-arms the identical failure at the next specifier change. The literal is the defect; the invariant is the fix.
- **Lowering the floor from 50 to 45.** The floor is the only reason casualty 3 surfaced at all. Tuning a non-vacuity guard down to match a degraded measurement converts a working alarm into a silent one — the exact shape of the four-spec smoke this ticket exists to complement.
- **Deleting the guards as post-split debris.** Their subjects are alive and correct; #10595's failure class is real and would become unguarded.
- **Filing this as part of #201.** #201 is blocked by four tickets. A repair that is unblocked today must not inherit their blocks — and #201's own AC list does not cover repairing a stale assertion.

## Related

- #201 — Run the retained Brain unit suite in CI. This ticket blocks it: three reds sit on the path it must bind.
- #194 — Make the retained Brain test suite real. Parent family.
- #89 — order-pollution class (closed); #191 / #193 / #195 — the other disposition owners named by #201.
- #250 — the §2.3/§2.7 lane this was found from; the same migration root, a different leaf.
- #10595 — the `ReferenceError: Neo is not defined` regression class casualty 1 guards.

## Handoff Retrieval Hints

Retrieval Hint: "structural guard asserts the pre-split relative specifier while the subject is package-qualified"
Retrieval Hint: "non-vacuity floor fired because the import walker skips bare package specifiers"
Retrieval Hint: `.github/workflows/brain-unit.yml` executes four named specs after `--list` collects 11771 tests in 762 files

Live latest-open sweep: checked the latest 20 open issues at 2026-08-31T03:12:33Z; no equivalent found (#201 is the CI-binding ticket, related and explicitly not duplicated here). A2A in-flight claim sweep at 2026-08-31T03:09Z over the last 40 messages, all read-states: no peer claim on this scope.

Origin Session ID: 0815f0e9-c789-4f98-8629-1356c6eae7eb

## Timeline

- 2026-08-31T03:14:11Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-31T03:14:12Z @neo-opus-grace added the `bug` label
- 2026-08-31T03:14:12Z @neo-opus-grace added the `ai` label
- 2026-08-31T03:14:12Z @neo-opus-grace added the `testing` label
- 2026-08-31T03:14:13Z @neo-opus-grace added the `regression` label
- 2026-08-31T03:14:13Z @neo-opus-grace added the `agent-os` label
- 2026-08-31T03:14:44Z @neo-opus-grace cross-referenced by #201
- 2026-08-31T03:26:37Z @neo-opus-grace referenced in commit `ac44fd4` - "fix(test): guards assert the Neo bootstrap, not one specifier (#271)

Three tests were red on dev at head while every Brain PR reported `unit ✓ pass`.
One root: ADR 0040 §2.3 moved the Agent OS onto the published Engine, so
`sweepExpiredTasks.mjs` and `syncGithubWorkflow.mjs` import `neo.mjs/src/**`.
Their guards still asserted the pre-split relative climb `../../../src/Neo.mjs`.
The production side of the migration landed; the guard side did not.

- match the module each import RESOLVES to, in any specifier form, so the next
  specifier migration does not silently un-guard #10595 the way this one did
- presence assertions carry their own messages: `findIndex` returns -1, and
  `-1 < lifecycleIdx` is vacuously true, so an ordering assertion reached with a
  missing import would report correct ordering about an import that is not there
- teach `reachesPackage` the package boundary: first-party subpaths
  (`neo.mjs/src/**`) are followed into node_modules; third-party bare specifiers
  stay leaves, which is how the package under test is detected. Reach from the
  stage entry recovers 45 -> 63
- stop short-circuiting the walk on hit. Both callers assert a non-vacuity floor
  on `walked` BEFORE they assert `reached`, so a real chromadb regression failed
  the FLOOR and told the reader to check that a file resolves. Verified: the
  mutant now fails the property, with the right diagnostic
- the floors are re-derived, not moved. 50 was already correct against 301
  (barrel) and 63 (stage entry); the calibration basis is now stated inline

Four mutants verified red for the right reason: prelude deleted from each
subject; module-scope chromadb re-introduced on the stage entry; the walk
truncated. No production file changes — the production side is correct.

Feeds #201, which cannot bind the retained suite over three reds."
- 2026-08-31T03:26:38Z @neo-opus-grace cross-referenced by PR #272
- 2026-08-31T03:50:11Z @neo-opus-grace referenced in commit `5744ea0` - "fix(test): resolve specifiers instead of matching their spelling (#271)

@neo-gpt found two false greens at ac44fd45c5, and both were the same defect:
the guards claimed to assert resolved module identity while the code compared
strings.

RA-1a — the prelude regex matched a SUFFIX. `['"][^'"]*src/Neo.mjs['"]` accepts
`fake/src/Neo.mjs`, so a subject whose bootstrap does not resolve at all read
3/3 green while direct invocation would still die before Neo.gatekeep(). Both
preludes now resolve through resolveModuleSpecifier() and compare against the
resolved target, so a specifier that does not resolve returns null and can
never match. A floor asserts the targets themselves resolve, because comparing
against null would make -1 === -1 read as agreement.

RA-1b — the forbidden-package detector compared `specifier === barePackage`,
seeing only the package root, so `import 'chromadb/subpath'` — reach into the
forbidden package by any reading — passed as clean. normalizeSpecifier() now
collapses subpaths before comparing, and handles the scoped-package case.

Both repairs delegate to ai/scripts/lint/scriptPlaneClosure.mjs rather than
maintaining a second resolution model beside it. That module was already the
repository's authority for exactly these two semantics; building a parallel
FIRST_PARTY_PACKAGES map next to it was the underlying mistake, not just the
two symptoms.

Both of @neo-gpt's exact mutations verified red against the repaired guards and
reverted: fake/src/Neo.mjs -> neoImportIdx -1; chromadb/subpath -> reached true.
Baselines green, 3/3 and 6/6."
- 2026-08-31T06:34:59Z @tobiu referenced in commit `7ec129d` - "Merge pull request #272 from neomjs/fix/structural-guards-pre-split-layout-271

fix(test): guards assert the Neo bootstrap, not one specifier (#271)"
- 2026-08-31T06:34:59Z @tobiu closed this issue

