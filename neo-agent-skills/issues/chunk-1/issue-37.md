---
id: 37
title: 'npm test short-circuits at the first runner, so three of five never execute'
state: OPEN
labels:
  - bug
  - ai
  - testing
  - build
  - model-experience
assignees: []
createdAt: '2026-09-01T22:34:33Z'
updatedAt: '2026-09-01T23:11:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/37'
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
---
# npm test short-circuits at the first runner, so three of five never execute

## Context

Operator, 2026-09-01, on the hand-rolled runners: *"the wannabe handrolled js tests are ugly"*, and *"while we very rarely use `npm test`, improvements make sense."*

The ugliness is the visible half. Measuring it turned up a defect: the suite does not report what it claims to report.

## The Problem

`package.json` wires the gate as a shell conjunction:

```json
"test": "node scripts/test-lint-skill-corpus.mjs && node scripts/test-reusable-pr-baseline.mjs && node scripts/test-ticket-archaeology.mjs && node scripts/test-substrate-size.mjs && node scripts/test-check-pr-body.mjs"
```

`&&` short-circuits. **The first non-zero exit ends the run**, so every later runner is skipped — not reported as skipped, simply absent. Measured on `dev` at `27ad895` (v0.1.3), running each file individually:

| runner | exit | reached by `npm test`? |
|---|---|---|
| `test-lint-skill-corpus` | **1** (`24/25`, materialization check FAILED) | yes — and it stops here |
| `test-reusable-pr-baseline` | **1** | **no** |
| `test-ticket-archaeology` | 0 | no |
| `test-substrate-size` | 0 | no |
| `test-check-pr-body` | 0 | no |

**Two runners are red on `dev`, not one, and the second is structurally invisible.** Its three assertions — `package version drift`, `substrate package version drift`, `PR-body package version drift` — are `#27`'s symptom (the baseline's `SKILLS_VERSION` pin), so the defect already has an owner; what has no owner is that its test has been failing where nobody can see it.

The failure mode is the one this repository's own guards exist to catch: a green-looking gate over unexecuted checks. `npm test` printing `24/25 passed` reads as a suite verdict. It is one file of five.

## The Architectural Reality

- Each runner re-implements a test framework by hand: `test-substrate-size.mjs` alone carries `fixture()`, `write()` and `capture()` helpers plus its own temp-dir bookkeeping, and the others repeat the pattern. Assertions come from `node:assert/strict`; everything around them is bespoke.
- There is **no per-test granularity**. A file is the unit, so one failing assertion takes its whole file's remaining arms with it — a second short-circuit inside the first.
- `engines` already declares `node >=24.0.0`. Node's built-in runner is stable there: `describe`/`it`, `node:assert/strict` unchanged, a spec reporter, name/pattern filtering, per-test results, and `--test-concurrency`.
- The tests are pure Node — `spawnSync` over the CLIs, `mkdtemp` fixtures, symlink-resolution arms. **No browser, no DOM, no shared harness with any sibling repository.**

## The Fix

Move the five runners to `node:test` and replace the conjunction with one invocation:

```json
"test": "node --test scripts/"
```

Every file runs, every failure is reported, and the exit code still fails the gate.

**Not Playwright.** The sibling repositories use it, so consistency is the obvious argument — and it buys the name rather than the machinery. `neomjs/neo`'s usage is custom configs per project built around browser fixtures and the Neural Link; these tests share none of that. Against it: this package is installed by **every** consumer repository and should stay the lightest thing in the organisation, and a browser-provisioning devDependency is weight bought for a test surface that never opens a page. `node:test` gives the structure the operator is asking for at zero dependencies.

## Acceptance Criteria

- [ ] `npm test` executes all five files in one run; a failure in the first does not prevent the rest from reporting. Demonstrated by making an early file fail deliberately and showing later files still report.
- [ ] Each runner's arms become individually named tests, so a single failing arm does not suppress its file's remaining arms.
- [ ] The gate still fails: a non-zero exit for any failing test, verified by a seeded failure.
- [ ] No new runtime or dev dependency; `package.json` `files` is unchanged, so nothing new reaches consumers.
- [ ] Every assertion that exists today survives the move — count the arms before and after and state both numbers; a migration that quietly drops coverage is worse than the short-circuit.
- [ ] The two failures currently red on `dev` are visible in one `npm test` run afterwards, rather than one hiding the other. Fixing them is out of scope; **seeing** them is the point.

## Out of Scope

- Fixing either red runner. `test-reusable-pr-baseline`'s version drift belongs to `#27`; `test-lint-skill-corpus`'s materialization arm needs its own diagnosis and may be environment-specific.
- Adopting Playwright here, or aligning runners across repositories — see the Fix's reasoning; if that is wanted it is an `#14` decision, not this leaf's.
- The CI workflows that call these scripts, beyond whatever the `test` script change requires.

## Avoided Traps

- **Replacing the conjunction with `;` or `||` and calling it fixed.** That makes every runner execute but throws away the exit code, converting a short-circuiting gate into no gate at all.
- **Porting the bespoke helpers verbatim into `node:test`.** The helpers exist because there was no runner; most of `fixture()` and `capture()` dissolve into `t.after()` and ordinary assertions.
- **Treating "24/25 passed" as a baseline to preserve.** That number is one file's, and the migration should make the real total visible even if it is uglier.
- **Adding a devDependency to a package every consumer installs.** The whole argument for `node:test` is that it costs nothing downstream.

## Related

- `#27` — the version-drift defect whose test has been failing invisibly (assigned, open)
- `#14` — the PR-governance Epic that owns cross-repository tooling shape
- `#24` — the preflight-reachability defect, same repository, different surface

**Live latest-open sweep:** latest 20 open issues in `neomjs/neo-agent-skills` read created-descending at 2026-09-01T22:33:24Z; a `test runner OR node:test OR playwright OR harness in:title` search across every state returned `#24`, `#8`, `#14` and two closed tickets, none equivalent. **A2A claim sweep:** last 15 messages, all read-states, back to 21:49Z — claims cover Engine popup drag (`#18054`), institution PRs `#70`/`#71`/`#72`, and seat re-keying; no overlap with this scope.

**Structure-map gate:** N/A — this modifies existing files under `scripts/` and adds no new `.mjs` placement.

**unowned-rationale:** left unassigned deliberately. The GPT bench is at 0% weekly quota and the Claude side is at reduced volume tonight; this is a repository-shape improvement with no consumer waiting on it, and the operator's own framing is *"we very rarely use `npm test`"*. It should be picked up when the shape work is being done, not ahead of a defect. Claimable by anyone.

Origin Session ID: 5e4492c2-ace7-47e8-83bf-98ccca3a684b

Authored by Grace (Anthropic Claude Opus 5, Claude Code).

## Timeline

- 2026-09-01T22:34:34Z @neo-opus-grace added the `bug` label
- 2026-09-01T22:34:35Z @neo-opus-grace added the `ai` label
- 2026-09-01T22:34:35Z @neo-opus-grace added the `testing` label
- 2026-09-01T22:34:35Z @neo-opus-grace added the `build` label
- 2026-09-01T22:34:35Z @neo-opus-grace added the `model-experience` label
- 2026-09-01T22:46:31Z @neo-fable cross-referenced by #38
### @neo-opus-grace - 2026-09-01T23:11:51Z

## Anchor correction: one of the two reds is fixed — and the way it got fixed is the ticket's own point

Filed at 22:34Z against `dev@27ad895` with the evidence table showing **two** runners red. Re-measured at `dev@7d1e4be`:

| runner | at `27ad895` (filing) | at `7d1e4be` (now) |
|---|---|---|
| `test-lint-skill-corpus` | exit 1 — `24/25`, materialization check FAILED (38) | **exit 1, unchanged** |
| `test-reusable-pr-baseline` | exit 1 — three `… version drift` assertions | **exit 0** — *"canonical contract + 42 negative mutations passed"* |

`#35` (Resolves `#27`) merged at 22:41:30Z and repaired the `SKILLS_VERSION` pin, which is what those three assertions were reporting. So the second red is gone, ~7 minutes after this ticket was filed.

**The ticket's substance does not change; the anchor gets sharper.** The point was never *"two runners are red"* — it was that `&&` means runners 2–5 never execute, so a failure in the second is invisible to `npm test`. That is still true at `7d1e4be`: the first runner still exits 1, so the other four still never run, and the suite still reports one file's `24/25` as if it were a verdict.

And the way this red cleared is the argument, not a counter-argument to it: **nobody fixed it because `npm test` surfaced it.** `#27` was found while porting a guard in PR `#26`, filed as its own defect, and repaired by `#35` — an entirely separate path. The gate that should have shown it red for however long the pin was wrong showed a green-looking `24/25` from a different file instead. A defect that gets fixed by luck of an adjacent lane is exactly the cost of a short-circuiting gate.

**AC impact:** the last acceptance criterion — *"the two failures currently red on `dev` are visible in one `npm test` run afterwards"* — should now read **one** failure (`test-lint-skill-corpus`) rather than two. Whoever picks this up: re-measure before writing the arm rather than trusting either number, since this one moved inside an hour.

Everything else stands, including that fixing `test-lint-skill-corpus`'s materialization arm remains out of scope here — it needs its own diagnosis and may be environment-specific.

🖖 Grace


- 2026-09-03T19:47:14Z @neo-opus-grace cross-referenced by #41
- 2026-09-03T21:32:37Z @neo-opus-grace cross-referenced by PR #42
- 2026-09-15T01:57:33Z @neo-opus-ada cross-referenced by PR #68
- 2026-09-18T10:07:45Z @neo-opus-ada cross-referenced by PR #89
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-20T02:19:48Z @neo-gpt-emmy cross-referenced by PR #98

