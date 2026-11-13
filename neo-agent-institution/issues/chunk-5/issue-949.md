---
id: 949
title: The visual stamp misreads a CRLF checkout and moves its engine line
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - build
assignees: []
createdAt: '2026-11-13T05:37:30Z'
updatedAt: '2026-11-13T06:21:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/949'
author: neo-opus-ada
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# The visual stamp misreads a CRLF checkout and moves its engine line

## Context

PR #938 (head `d9c6190b`, an outside contributor on Windows) restamped `test/playwright/visual/__screenshots__/baseline-inputs.txt` after a source-only change, with no golden moved. The stamp's `engine` line still moved, from `41e5632…` to `781af05…`, the contributor's installed engine. #852 (PR #853) closed this trap; a CRLF checkout reopens it.

## The Problem

Design authority: #852 and the stamp header in `buildScripts/checkVisualBaselines.mjs`: "`engine` is the revision of the last stamp that changed a golden".

`parseStamp` splits the file on `\n` and trims values, but not keys. Git for Windows checks text out with CRLF by default (`core.autocrlf=true`), and this repository has no `.gitattributes`. On such a checkout every key keeps a trailing `\r`:

1. `entries.get('engine')` misses (`engine\r`), so `previous.engine` is `null` and `nextStampEngine` takes its first-stamp branch: `--stamp` records the running engine.
2. `diffStamp` finds no key in common, so `check-visual-baselines` fails with every input drifted.

The serializer writes clean LF lines, so the stamp's diff shows only the engine line, which is easy to miss in review.

Reproduction (pure functions, nothing written): parse dev's stamp at `1b446cd` as is and with `\n` → `\r\n`.
- `nextStampEngine(previous, current)`, with PR #938's entries and engine `781af05…`: LF keeps `41e5632…`; CRLF returns `781af05…`, and `parseStamp(crlf).engine` is `null`.
- `diffStamp` against the stamp's own entries: LF reports 0 of 489 inputs; CRLF reports 979 rows (`engine\r (removed)`, then each input as removed and added).

## The Architectural Reality

- `buildScripts/checkVisualBaselines.mjs`: `parseStamp(text)` runs `text.split('\n')`, sets `key = line` and `entries.set(key, line.trim())`. `diffStamp` and `nextStampEngine` (#853) consume its result. The main flow reads the stamp from the working tree with `fs.readFileSync(…, 'utf8')`.
- `listingEntries` parses `git ls-files -s` output, which git writes with LF on every platform, so only the stamp file's own read is affected.
- The pure halves are import-safe (the main flow runs only as the entry script), so a spec needs no git.

## The Fix

1. `parseStamp` splits on `/\r?\n/`, so a CRLF checkout parses exactly like an LF one.
2. Specs beside #853's: a stamp and its CRLF copy parse to the same `{engine, entries}`; on the CRLF copy, with no golden changed, `nextStampEngine` keeps the previous engine and `diffStamp` reports nothing.

## Acceptance Criteria

- [ ] `parseStamp` returns the same `{engine, entries}` for a stamp and its CRLF copy (unit).
- [ ] With no golden changed, a CRLF stamp keeps its `engine` through `nextStampEngine`, and `diffStamp` against matching entries is empty (unit).
- [ ] `npm run check-visual-baselines` passes on dev.

## Out of Scope

- Capturing goldens off macOS (they are Darwin captures; #864).
- PR #938's engine line, which its review restores by hand.

## Avoided Traps

- **A `.gitattributes` rule alone.** It fixes checkouts made after it lands; a contributor's existing clone keeps CRLF until the file is checked out again. The parser fix works on any checkout.
- **Trimming whole lines.** Values are already trimmed; trimming keys would also accept stray spaces in a hand-edited stamp. Splitting on `\r?\n` removes only the line ending.

## Related

#852 and PR #853 (the guard this repairs) · #864 (macOS goldens) · PR #938 (the occurrence)

Live latest-open + closed sweep: latest 20 open (60 open, red band; none on the stamp) plus closed searches ("visual stamp engine", "CRLF OR autocrlf OR line endings") at 2026-11-13T05:38Z; #852's Out of Scope and Avoided Traps read, neither excludes this. A2A in-flight claim sweep: latest 30, all read states; no claim on this scope. Memory Core sweep ("visual baseline stamp engine line moved on a Windows contributor restamp, CRLF line endings"): no prior decision. Own-assignment sweep: #516, #424, neither on this surface. No open ticket owns the stamp, so this files standalone.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The specs run without macOS.

Origin Session ID: 2c72d244-12b7-4acc-9cf4-d81507e74acb

Retrieval Hint: "visual stamp CRLF checkout: engine line moves on restamp, check reports every input drifted, Windows contributor"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T05:37:31Z @neo-opus-ada added the `bug` label
- 2026-11-13T05:37:31Z @neo-opus-ada added the `good first issue` label
- 2026-11-13T05:37:31Z @neo-opus-ada added the `help wanted` label
- 2026-11-13T05:37:32Z @neo-opus-ada added the `ai` label
- 2026-11-13T05:37:32Z @neo-opus-ada added the `build` label
- 2026-11-13T05:38:16Z @neo-opus-ada cross-referenced by PR #938
### @IG-ABHINAV - 2026-11-13T06:20:57Z

Claiming #949.

Reproduced on this Windows checkout (git core.autocrlf=true, `git ls-files --eol` reports `i/lf w/crlf` for `baseline-inputs.txt`): parsing the stamp's CRLF text returns `engine = null` (LF returns `41e5632698be771e8255859f7cb7cac6086894e7`), `nextStampEngine` falls into its first-stamp branch and records the running engine `781af05`, and `diffStamp` against the stamp's own entries reports 979 rows (`engine\r (removed)` first).

Fix in flight: `parseStamp` splits on `/\r?\n/` plus the unit specs from the issue's acceptance criteria, beside #853's.

### @github-actions - 2026-11-13T06:21:10Z

Thanks for claiming this, @IG-ABHINAV, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@ig-abhinav -->

- 2026-11-13T07:02:18Z @IG-ABHINAV cross-referenced by PR #952

