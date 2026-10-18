---
id: 1094
title: A pin-drift report names the engine fixes behind a consumer's pin
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T02:02:58Z'
updatedAt: '2026-10-18T03:12:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1094'
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
closedAt: '2026-10-18T03:12:04Z'
---
# A pin-drift report names the engine fixes behind a consumer's pin

## Context

- **2026-10-18, a workaround for a fix that already existed.** Brain #1092 added `cloneConfigSubtree`, which walks `Provider` parents because `Provider#getTopLevelDataKeys` enumerates own keys only at the Brain's engine pin `7f16355a` (2026-09-25). neo #19269 (`a87a89e2`, 2026-09-26) had already fixed exactly that upstream. Nothing told the author; a reviewer found it by hand (review 5560769338, RA-3).
- **2026-09-30, an installed engine the lock does not name.** An Institution checkout ran its visual suite on an engine that differed from its lock, and a false defect-note followed. Institution #340 now guards that one suite.
- **2026-10-18, the same state again.** My Institution checkout's lock names `2d3e6fa8`, while `node_modules/neo.mjs` is `9e481352`, so its unit and e2e runs load an engine the lock does not name.

## The Problem

Consumers pin the engine by commit, and neither question has a mechanical answer: *which engine am I running?* and *what between it and `dev` touches what I load?* Measured on 2026-10-18 against neo `origin/dev`, with a scratch probe (0.9 s):

| Consumer | Pin | Behind | Fixes | Fixes touching files it loads |
|---|---|---|---|---|
| neo-agent-brain | `7f16355a` (09-25) | 312 | 109 | 22, #19269 among them |
| neo-stage | `e8af9f3c` (10-11) | 121 | 37 | 7, e.g. #19838 in `test/playwright/fixtures.mjs` |

Nobody reads 109 fix subjects before writing a workaround. The 22 are the ones that can matter, and they are computable.

## The Architectural Reality

- **The pin** is `package-lock.json` → `packages["node_modules/neo.mjs"].resolved`, either `…/archive/<sha>.tar.gz` (Brain) or `git+ssh://…#<sha>` (Stage, Institution). **What npm installed** is the same key in `node_modules/.package-lock.json`; #340 compares the two the same way.
- **What a consumer loads** starts at its own imports into the engine: `neo.mjs/…` specifiers, relative paths into `node_modules/neo.mjs/`, and HTML module scripts (`src/MicroLoader.mjs`). From there, the engine's relative imports at the pin complete the closure.
  - Workers are not imports: `src/worker/Manager.mjs` spawns them by file name, so a MicroLoader root adds the top-level `src/worker/*.mjs` entries.
  - A computed `import()` cannot be followed: Brain has 103 and the closure 3. The report counts and lists them instead of hiding the gap.
- **History** needs an engine git checkout. Every seat has one beside its other clones (`neomjs/neo`). The GitHub compare API caps at 250 commits and omits per-commit files, so it cannot answer for Brain's 312.
- **Sibling:** `ai/scripts/diagnostics/consumerRelevanceCensus.mjs`. It is a read-only `git log --name-only` census with exported helpers, a `main()` behind the entry guard, and no AiConfig. The structure map (`ai:structure-map`, 2026-10-18) lists 34 files in `ai/scripts/diagnostics`.

## The Fix

Add `ai/scripts/diagnostics/pinDrift.mjs` and `npm run ai:pin-drift`.
1. Read the pin, the installed sha, and the `package.json` spec verbatim (a moving `#dev` is visible as such).
2. Walk the closure at the pin through one `git cat-file --batch` per level.
3. Run `git log --name-only <pin>..<ref>` and keep the commits that touch the closure. Print fixes first, each with its touched files and the chain to the consumer file that reaches them.
4. It is read-only: no fetch, no install, no write. A pin missing from the engine's history fails with the fetch command, and the ref's date is printed so a stale `--ref` is visible.
5. Pure helpers are exported for the spec: pin parsing, specifier mapping, the closure walk over an injected reader, and commit intersection.

No deletion: the report replaces a hand census (`git diff --stat` over the pin range), which no committed file records.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `--consumer <path>` | consumer checkout | the repo to report on | this Brain checkout | script JSDoc | AC-6 |
| `--engine <path>` | engine git checkout | history and closure source | the consumer's sibling `neo`, accepted only when its package name is `neo.mjs`; otherwise exit 1 naming the flag | script JSDoc | AC-6 |
| `--ref <rev>` | engine git | comparison target | `origin/dev` | script JSDoc | AC-4 |
| `--module <path>` (repeatable) | engine paths | commits touching that path, inside the closure or not | none | script JSDoc | AC-4 |
| text report | stdout | identity, pin and installed sha, counts, unfollowed imports, commits (fixes first) | none | script JSDoc | AC-7 |
| `--json` | stdout | the same facts as one object | none | script JSDoc | AC-7 |
| exit code | process | 0 report; 1 usage or environment (no lock entry, no engine, pin not in history) | none | script JSDoc | AC-1, AC-6 |

## Decision Record impact

None. No AiConfig read or write: flags only (critical gate 10 checked).

## Acceptance Criteria

- [ ] **AC-1** Unit: both lock forms yield the 40-hex pin. A lock without the `node_modules/neo.mjs` entry, or without a sha, refuses and names the field.
- [ ] **AC-2** Unit: `neo.mjs/x` and `../../node_modules/neo.mjs/x` map to `x`, and an HTML module script into the engine is a root. A JSDoc line (` * import A from '../A.mjs'`) is not an import. `import(name)` is counted as unfollowed, with its file.
- [ ] **AC-3** Unit: over an injected reader, the walk follows relative imports transitively, terminates on cycles and reports missing targets. A `src/MicroLoader.mjs` root adds the top-level `src/worker/*.mjs`.
- [ ] **AC-4** Unit: of two commits, only the one touching a closure file is reported, and fixes print first. `--module` reports a commit touching that path outside the closure.
- [ ] **AC-5** Unit: installed against lock, in three cases. Equal; different, where the report prints both shas and `rm -rf node_modules/neo.mjs && npm install` (#340's realign); and no hidden lock, reported as not installed.
- [ ] **AC-6** Live, on a Brain checkout whose lock pins `7f16355a`: the report lists `a87a89e2` under `src/state/Provider.mjs` ← `ai/ConfigProvider.mjs`. On a neo-stage checkout whose lock pins `e8af9f3c`, it lists #19838 under `test/playwright/fixtures.mjs`. In the JSON, every listed commit carries at least one closure file.
- [ ] **AC-7** The Brain text report stays within 60 lines, and `--json` parses with the same counts.
- [ ] **AC-8** The script's JSDoc (`@summary` and usage) and the `ai:pin-drift` npm script.

## Out of Scope

- Moving any pin.
- Failing test runs on an installed/lock mismatch. That is #340's per-suite guard shape, and a general pre-test guard is a follow-up once this report exists.
- CI scheduling, an MCP tool, skill wiring, and symbol search (`git log -G`).

## Avoided Traps

- **The GitHub compare API:** a 250-commit cap and no per-commit files.
- **A third-party parser:** statement-start regexes suffice for a relevance filter that errs toward inclusion, and the engine's own source is the parser's only input.
- **Path-prefix filtering** (`src/state/`) instead of a closure: it misses transitive files (`src/mixin/VdomLifecycle.mjs` reached through `src/component/Base.mjs`) and over-includes siblings.
- **Fetching inside the tool:** it is read-only like its siblings, and the printed ref date shows staleness.

## Related

- Brain #1092: review 5560769338, RA-3.
- neo #19269.
- Institution #340.
- D#10137, comment 19009485 (MX item 2): context, not authority.

Live latest-open sweep: the latest 20 open neo-agent-brain issues, checked at 2026-10-18T02:02:42Z; no equivalent found.
A2A claim sweep: the last 30 messages, all read-states; no pin-drift claim.
MC sweep: "Brain neo pin predates the engine fix, workaround re-implemented in consumer…", 6 results; no prior decision found. `gh search issues --owner neomjs` on "pin drift", "engine pin", "installed engine", "pin move" and "stale node_modules" surfaced #340 (cited) and pin-move tickets; no report exists.
Own-assignment sweep: 15 open in Brain; #121 (process freshness) read, no overlap.

Origin Session ID: 17da944b-3daa-48b5-8d16-c3c65a9f4ef4
Retrieval Hint: "consumer engine pin drift fixes behind pin closure"; "cloneConfigSubtree getTopLevelDataKeys pin 7f16355a #19269"


## Timeline

- 2026-10-18T02:02:59Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T02:02:59Z @neo-opus-ada added the `enhancement` label
- 2026-10-18T02:03:00Z @neo-opus-ada added the `ai` label
- 2026-10-18T02:03:00Z @neo-opus-ada added the `model-experience` label
- 2026-10-18T02:14:24Z @neo-opus-ada cross-referenced by PR #1095
- 2026-10-18T03:12:04Z @tobiu closed this issue
- 2026-10-18T03:12:04Z @tobiu referenced in commit `86fba3e` - "feat(diagnostics): a pin-drift report names the engine fixes behind a consumer's pin (#1094) (#1095)

npm run ai:pin-drift reads a consumer's engine pin and its installed engine, walks the engine
files the consumer loads at the pin, and lists the engine commits since the pin that touch them,
fixes first. --module asks the same question for one engine path, loaded or not.

Origin: #1092 re-implemented neomjs/neo#19269 against a pin one day older, and
neomjs/neo-agent-institution#340 caught a suite running an engine its lock did not name."
- 2026-10-18T04:24:24Z @neo-opus-ada cross-referenced by #1105

