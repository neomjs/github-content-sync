---
id: 412
title: 'README count arms pin the literal 40: a 41st skill turns five red'
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - testing
assignees: []
createdAt: '2026-11-16T18:44:43Z'
updatedAt: '2026-11-16T18:44:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/412'
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
blockedBy:
  - '[x] 363 The README''s two skill counts go stale with the next skill'
blocking: []
---
# README count arms pin the literal 40: a 41st skill turns five red

Please comment to claim before starting; check current assignees and linked PRs.

## Context

PR #408 (Resolves #363) adds a lint arm that compares the README opening's two skill counts with `skills.manifest.json`, and six fixture arms for it in `scripts/test-lint-skill-corpus.mjs`. This ticket is the follow-up from that PR's second review round. The first round's required action itself offered the literal as the search key, so this is the reviewer's follow-up, not an author miss.

Work starts once #408 is merged: the arms do not exist on `dev` before that (`blocked_by` #363).

## The Problem

The fixture arms carry the current count as a literal. `readmeCountLines()` searches for `40 skills`, three arms replace `'40 skills:'`, `', 40 skills'` and `'40 skills'`, and `addFortyFirstSkill` assumes the corpus holds 40.

Reproduced on #408's head `0f3cad1`, in a throwaway copy: add a real extra skill (its directory and its manifest row) and update both README counts to 41. That is the correct change for a new skill. The lint passes, and the suite fails five arms:

```
$ node scripts/lint-skill-corpus.mjs
skill-corpus: 41 skills, coherent with the manifest, within budget, document reach canonical.

$ node scripts/test-lint-skill-corpus.mjs
  FAIL 41 skills in manifest with unchanged README fails naming README.md and both count lines → exit 1 (expected 1)
  FAIL 41 skills in manifest with only the first README count updated still fails naming the subset count line → exit 1 (expected 1)
  FAIL 41 skills in manifest with both README opening counts updated passes → exit 1 (expected 0)
  FAIL removing a skill count from the README opening fails naming README.md → exit 0 (expected 1)
  FAIL inserting lines above the README opening counts shifts reported lines at 41 skills → exit 1 (expected 1)
```

So the next pull request that adds a skill correctly gets a red `npm test` for a reason unrelated to its change, and its author has to rename literals in the suite before it turns green.

## The Architectural Reality

- `scripts/test-lint-skill-corpus.mjs`: `addFortyFirstSkill()`, `readmeCountLines()`, and the six cases under `// ── README opening skill counts`.
- `run()` copies the live `README.md` and `.agents/` into a temp tree for each case, so the live manifest's count is available to the suite: `Object.keys(manifest.skills).length`.
- The lint arm (`scripts/lint-skill-corpus.mjs`, §3c) is correct and stays as it is.

Design authority: "Pasting a number into prose with no SSOT link (it will rot)." (`.agents/skills/neo-identity-update/references/update-protocol.md`, Anti-patterns). The same holds for a count pasted into a test.

## The Fix

In `scripts/test-lint-skill-corpus.mjs` only:

- read the count `n` once from the live manifest;
- build every search string, replacement string and regular expression of the README arms from `n` and `n + 1`;
- rename `addFortyFirstSkill` to say what it does (it adds one skill).

## Acceptance Criteria

- [ ] AC-1: no numeric literal for the current skill count remains in the README count arms or their helpers.
- [ ] AC-2: `node scripts/test-lint-skill-corpus.mjs` passes on `dev` with every existing arm still present.
- [ ] AC-3: with a real extra skill added (directory, manifest row, both README counts updated), the lint passes and the suite passes. The pull request body carries that run as its receipt.

## Out of Scope

- The lint arm and its messages.
- The README's wording.
- The version in `package.json`: a fork leaves it as it is (#358 explains why).

## Avoided Traps

- **Reading the count from the README.** The README is the thing under test; the manifest is the index it describes.
- **A pattern that accepts any number.** The arms must still prove that the lint names the right line for the right count.

## Related

#363 (the guard's ticket; this one is `blocked_by` it) · #408 (the pull request that adds the arms)

unowned-rationale: Ready for a contributor; maintainers provide planning and review. It needs no private tool: one file, and the receipt is two commands.

Live latest-open + closed sweep: checked the latest 20 open plus an open and closed search (`README count arms literal 40 test-lint-skill-corpus`, `README skill count test lint-skill-corpus`) at 2026-11-16T18:44Z; only #363 matches, and it is the guard's own ticket and #408's close target. 38 open: orange band; no owning epic exists for the corpus lint, so this is filed as a leaf.
MC sweep: "test suite turns red when a new skill is added because README count arms pin the number 40"; no prior decision found.
Own-assignment sweep: #5, #6 and #302, none on this surface.
A2A sweep: the 30 newest messages at 18:41Z; no claim on this scope.
Structure map: N/A (one test script under `scripts/`, no skill file touched).

Origin Session ID: 9255df3c-9e74-4809-b358-112c6aac837e
Retrieval Hint: "README count arms literal 40", "test-lint-skill-corpus 41st skill"

— Vega (Claude Fable 5.2, Claude Code) 🌿

## Timeline

- 2026-11-16T18:44:44Z @neo-opus-vega added the `bug` label
- 2026-11-16T18:44:44Z @neo-opus-vega added the `good first issue` label
- 2026-11-16T18:44:44Z @neo-opus-vega added the `help wanted` label
- 2026-11-16T18:44:44Z @neo-opus-vega added the `ai` label
- 2026-11-16T18:44:44Z @neo-opus-vega added the `testing` label
- 2026-11-16T18:44:52Z @neo-opus-vega marked this issue as being blocked by #363
- 2026-11-16T18:45:26Z @neo-opus-vega cross-referenced by PR #408

