---
id: 1105
title: 'A review range''s delta digest: files, symbols, tests, close targets'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T04:24:23Z'
updatedAt: '2026-10-18T04:57:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1105'
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
closedAt: '2026-10-18T04:57:57Z'
---
# A review range's delta digest: files, symbols, tests, close targets

## Context

Item 3 of the MX list on neomjs/neo#10137 (comment 19009485): summaries drift from diffs.

On #1086's Round 2, the author's review-response A2A described an RA-1 that is not in its delta. It names removed `config = AiConfig` defaults on `resolveSnapshot(config)` / `embedText(text, config)`, `?.` chains on `src`, and a new `embedText('probe', {})` test. None of those identifiers exists in `b4091753..ab49787a`, and the retry spec is untouched. It also says the body moved to `Part of #541`, while the live body read `Resolves #541`. The PR body and the commit were right; the prose came from recall.

Reviewers re-derive the same facts by hand. #1099's Round 2 needed `git log`, `--remerge-diff` on its dev merge, per-file `git show` and a caller census. Stage #196's reviewer asked the author for exact-head coordinates before reading anything.

## The Problem

A review range has a fixed set of facts:
- which paths changed, and how;
- which symbols appeared, vanished or changed;
- which test titles were added or removed;
- which tickets the range closes;
- whether a merge carried hand edits.

Authors restate these from memory, and reviewers re-derive them. Nothing states them once.

Past misses set the requirements:
- An enumerated path list "proved" an empty delta because `docs` was not in it (neomjs/neo#17324). The census starts from every path and subtracts only named generated trees.
- A two-dot diff on a rebased branch counted dev's arrivals as the PR's own (neomjs/neo#16674). The base is the merge-base.
- GitHub's `changedFiles` was read as "removed" (neomjs/neo#19497). Status and counts come from git, with rename detection.
- A grep for a method name missed it because the method was `async` (neomjs/neo#17362). Symbols come from a parser.
- A moved head was re-verified by re-reading the whole delta, when the cheap question was whether the reviewed files changed at all (#17362). A review anchor needs a "changed since" view.

## The Architectural Reality

- `get_pull_request_diff` (github-workflow MCP, `PullRequestService.getPullRequestDiff`, `ai/services/github-workflow/PullRequestService.mjs:3892`) returns GitHub's view. `files_only` gives path, additions and deletions; `file` gives hunks; `sha` pins a head. It is hermetic by design (neomjs/neo#10748), and it sees no merge-base, rename status, symbols, tests or merge resolutions.
- `ai/scripts/diagnostics/pinDrift.mjs` (#1094) is the sibling: a git-only report run from a Brain checkout against any repository. It has a private `git` runner, a `git cat-file --batch` reader (`readAtCommit`) and an entry-guarded `main()`.
- `acorn` is already a Brain dependency. It is used by `ai/scripts/lint/check-aiconfig-test-mutation.mjs`, `ai/scripts/lint/lint-config-template-ssot.mjs` and `ai/scripts/diagnostics/mcpHandlerSignatureCensus.mjs`.

## The Fix

1. Add `ai/scripts/diagnostics/deltaDigest.mjs`, run as `npm run ai:delta-digest -- [--repo <path>] [--base <ref>] [--head <ref>] [--since <sha>] [--generated <glob>]… [--json]`. The digest has seven sections:
   - **range:** base, merge-base and head; the non-merge commits; the merges, each marked when its `git show --remerge-diff` is non-empty;
   - **files:** `--name-status -M` and `--numstat` from the merge-base, binaries marked, and paths matching a `--generated` glob counted apart;
   - **symbols:** acorn over each JS file at both ends, listing what was added, removed or changed (same name, different source);
   - **tests:** `test` / `describe` / `it` titles by describe path, added and removed;
   - **close targets:** GitHub closing keywords before `#N` or `owner/repo#N` in the range's messages, plus `(#N)` subject ids;
   - **comments:** each authored file's added code, comment and blank lines, flagging files with more comment lines than code lines;
   - **since:** with `--since`, the files changed after that anchor, split into those touched by own commits and those touched only by merges.
2. Move the `git` runner and the batch reader out of `pinDrift.mjs` into `ai/scripts/diagnostics/gitObjects.mjs`, imported by both, so the second git diagnostic removes the first one's private copy instead of duplicating it.
3. Export pure functions and keep the CLI behind the entry guard. Output is text by default; `--json` carries the same facts for tools.

## Contract Ledger

| Surface | Authority | Behavior | Fallback / refusal | Docs | Evidence |
|---|---|---|---|---|---|
| `npm run ai:delta-digest` | this ticket | Prints the digest of `--base`…`--head` (defaults `origin/dev`, `HEAD`) for `--repo` (default: the cwd's repository) | An unresolvable ref exits 1 naming its flag; `--help` prints usage | module JSDoc, `USAGE` | unit + git fixture |
| `deltaDigest.mjs` exports: `buildDigest`, `formatDigest`, `extractSymbols`, `extractTestTitles`, `closeTargets`, `classifyAddedLines` | this ticket | Pure over strings and records, except `buildDigest`, which reads git | A file that does not parse at either end is reported as unparsed, with acorn's message | JSDoc `@summary` | unit |
| `gitObjects.mjs`: `git`, `readAtCommit` | moved from `pinDrift.mjs` | Unchanged | Unchanged | JSDoc | `PinDrift.spec.mjs` unchanged and green |

## Decision Record impact

none

## Acceptance Criteria

- [ ] **AC-1 Range.** `--base` and `--head` resolve to SHAs, and the digest diffs from their merge-base. It names base, merge-base and head, and lists the range's non-merge commits (sha, subject). It lists its merges, marking each merge whose `--remerge-diff` is non-empty.
- [ ] **AC-2 Files.** Every path changed in the range appears, with its status (added, modified, deleted, renamed with similarity, copied, type change) and its added and removed lines; binaries are marked. A `--generated` glob moves matching paths into a counted generated group. Nothing narrows the census to an include list.
- [ ] **AC-3 Symbols.** For `.mjs`, `.js` and `.cjs` files, acorn compares, between merge-base and head:
  - top-level declarations and exports;
  - class members, including static, async, getters, setters, private `#` names and fields.

  The digest lists added, removed and changed names. A file that fails to parse at either end is listed as unparsed with the parser's message, never skipped.
- [ ] **AC-4 Tests.** In `*.spec.mjs` and `*.test.mjs` files, titles of `test`, `test.describe`, `test.skip`, `test.only`, `describe` and `it` calls are compared by describe path. Titles come from string literals and from template literals without expressions. The digest lists added and removed titles.
- [ ] **AC-5 Close targets.** The range's commit messages are scanned for GitHub's closing keywords followed by `#N` or `owner/repo#N`, and each hit is listed with its commit. Subject ids `(#N)` are listed too.
- [ ] **AC-6 Comments.** For each authored file, the digest counts added lines as code, comment or blank, and flags a file whose added comment lines outnumber its added code lines.
- [ ] **AC-7 Since.** `--since <sha>` lists the files changed between that anchor and head. They are split into files touched by the range's own commits after the anchor and files touched only by merges.
- [ ] **AC-8 Output.** The default output is a compact text digest; `--json` prints the same facts as one object. `npm run ai:delta-digest` runs it, `--help` prints usage, and an unresolvable ref exits 1 naming its flag.
- [ ] **AC-9 Evidence.** A unit spec covers the pure functions. It also runs the CLI over a temp git fixture holding:
  - a rename, a delete and a binary file;
  - an async method and a private method;
  - a renamed test title;
  - a merge with a hand resolution, and one without;
  - a generated glob and a closing keyword.

  `PinDrift.spec.mjs` stays green on the shared `gitObjects.mjs`.

## Out of Scope

- **An MCP tool** wrapping the digest. Trigger: a seat without a shell needs it; the pure exports make the wrapper thin.
- **Skill or template changes** asking authors and reviewers to attach a digest (neo-agent-skills). Trigger: the digest has carried real reviews and its shape held.
- **A claim checker** that flags review prose naming identifiers absent from the digest. Trigger: the symbol and title sets prove stable across real ranges.
- **`--pr N`** resolution through `gh`. The caller fetches refs.
- **Symbols for non-JS files.** Those files appear in the files section only.

## Avoided Traps

- **A digest mode on `get_pull_request_diff`.** Symbols, tests and remerge-diffs need both trees and git's merge machinery, which a hermetic API call fetches file by file. The MCP tool stays GitHub's view, and this report is git's.
- **Hunk-header names** (`@@ … funcname`). They come from git's heuristic, which is wrong inside JS classes.
- **A list of "reviewed paths"**, or a two-dot base. See The Problem.

## Related

- neomjs/neo#10137 (comment 19009485, item 3): the source.
- #1086: the observed drift.
- #1094 and #1095: the sibling diagnostic.
- neomjs/neo#10748: `get_pull_request_diff`.
- neomjs/neo-stage#196: the exact-head evidence request.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T04:23:03Z; no equivalent found.
A2A claim sweep: the last 30 messages, all read-states, at 2026-10-18T04:20Z; no claim on a digest. Maryam's neo-agent-skills#12 is the skills-side instrument audit, not a tool.
MC sweep: "review response described identifiers that appear nowhere in its delta; digest of a commit range", no prior decision; the five lessons in The Problem came from it.
KB ticket sweep: `get_pull_request_diff` (neomjs/neo#10748) is the nearest; no digest.
Own-assignment sweep: nothing open on this surface.
Structure map: `ai/scripts/diagnostics`, sibling `pinDrift.mjs`; exit 0.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "delta digest of a review range: files, symbols, test titles, close targets"; "summaries drift from diffs, MX item 3"

## Timeline

- 2026-10-18T04:24:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T04:24:25Z @neo-opus-ada added the `enhancement` label
- 2026-10-18T04:24:25Z @neo-opus-ada added the `ai` label
- 2026-10-18T04:24:25Z @neo-opus-ada added the `model-experience` label
- 2026-10-18T04:39:55Z @neo-opus-ada cross-referenced by PR #1107
- 2026-10-18T04:57:57Z @tobiu referenced in commit `f4671a2` - "feat(diagnostics): a delta digest states what a review range changed (#1105) (#1107)

npm run ai:delta-digest reads a base…head range from git, measured from the merge-base: its commits and merges (marking a merge whose remerge-diff is not empty), every path with status and counts (named generated globs counted apart), the symbols and test titles acorn finds added, removed or changed at both ends, the tickets its messages close, each code file's added comment and code lines, and, with --since, what changed after a review anchor.

pinDrift's git runner and cat-file reader move to gitObjects.mjs, which both diagnostics import."
- 2026-10-18T04:57:57Z @tobiu closed this issue

