---
id: 346
title: The AGENTS.md pointer and its git filter live beside the generator
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T14:48:49Z'
updatedAt: '2026-11-11T17:32:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/346'
author: neo-opus-grace
commentsCount: 0
parentIssue: 335
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-11T17:32:15Z'
---
# The AGENTS.md pointer and its git filter live beside the generator

## Context

#335 made a consumer's `AGENTS.md` generated from its installed skills package. The Engine shipped the storage half first (neomjs/neo#20749, merged Nov 11): git keeps a three-line pointer in `AGENTS.md`, and every installed working copy holds the generated rules in that same file, behind a git filter. The mechanism lives in the Engine:
- `buildScripts/util/agentsMdPointer.mjs`: the pointer text, and a CLI that is the filter's clean command.
- `buildScripts/util/prepare.mjs`: `ensureAgentsMdFilter` registers the filter, and `refreshAgentsMd` lets the index re-read the file without staging content.

`devindex` is the second consumer (neomjs/devindex#135). Its intake decided that the mechanism moves here, beside the generator, before `devindex` adopts it (neomjs/devindex#135, comment 6798950302).

## The Problem

A copy per consumer means a defect per consumer. The Engine's copy shipped one. The filter was registered as `required` with no smudge, so every checkout that writes `AGENTS.md` died: a new worktree, a branch switch, a pull (neomjs/neo#20776, fixed by neomjs/neo#20777). A `devindex` copy would have carried the same defect and needed the same fix.

The Brain's Fleet also needs the pointer's first line, to tell a failed generation from the rules (neomjs/neo-agent-brain#1606). It copies the line today, because the Brain pins the Engine at a commit older than the pointer, while it already depends on this package.

## The Architectural Reality

Design authority: "Generated copies (`AGENTS.md` sections, `.claude/CLAUDE.md`) are never hand-edited; they regenerate from the package." (#140, the authority #335 cites). The pointer and its filter are how that copy is stored, so they belong with the package that regenerates it.

- `scripts/generate-agents-md.mjs` (bin `neo-agent-skills-agents-md`) writes the rules a consumer declares: `--repo <set> --out AGENTS.md`. The Engine runs it through its `generate-agents-md` script, from `prepare` and from every seat Start.
- `neo-agent-skills/agents-md` already exports `generate` and `readSupported` (the Brain imports them).
- The Engine's pointer, byte for byte, is three lines and a newline:
  - `<!-- neo-agents-md: pointer -->`
  - the "this file is generated" line;
  - the fork contributor's `npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo --audience contributor` line.
- The filter registration that works, after neomjs/neo#20777: `filter.neo-agents-md.clean` (the CLI), `filter.neo-agents-md.smudge = cat`, and `filter.neo-agents-md.required = true`. Each consumer's `.gitattributes` routes `/AGENTS.md` through `filter=neo-agents-md`.
- Git runs a filter command from the top of the working tree, through `sh`, and `node_modules/.bin` is not on its path. The clean command therefore names the module by path: `node node_modules/neo-agent-skills/scripts/<module>.mjs …`.

## The Fix

1. A module beside the generator exports the marker and the pointer text for a repository. It is the clean filter's CLI: it drains stdin and prints the pointer. It is reachable as `neo-agent-skills/agents-md/pointer`.
2. The registration (clean, smudge, required) and the index refresh move here as functions that take an injectable `git`, as in the Engine.
3. One call is the consumer's whole stage: register the filter, generate the rules into `--out`, and refresh the index. The consumer's `prepare` or `postinstall` runs that one bin, so the Engine and `devindex` each declare one line instead of carrying a copy.
4. The Engine's real-git arms move with the code, together with neomjs/neo#20777's checkout arm.
5. The package version is bumped in the PR, to a minor, because this adds a surface.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-skills/agents-md/pointer` (new export) | this ticket; the pointer shape of #335 and neomjs/neo#20749 | exports the marker and the pointer text for a repository | none; a consumer that does not adopt it keeps its own copy until it does | `README.md` (`## Generated AGENTS.md`, `## What is in the package`) and the module's JSDoc | a spec arm: the text for `neo` equals neomjs/neo's committed `AGENTS.md`, byte for byte |
| `neo-agent-skills-agents-md --pointer` | this ticket | registers clean, smudge and required; writes the rules; refreshes the index | a registration that cannot be established stops before the file is touched, as in the Engine | `README.md` (`## Generated AGENTS.md`) and the `run` JSDoc in `scripts/generate-agents-md.mjs` | the real-git arms |

## Acceptance Criteria

- [ ] `neo-agent-skills/agents-md/pointer` exports the marker and the pointer text. The text for `--repo neo` equals neomjs/neo's committed `AGENTS.md`, so the Engine's adoption changes no committed file.
- [ ] One bin call registers clean, smudge and required, writes the rules and refreshes the index. Afterwards `git status` is clean and the index holds the pointer.
- [ ] Real-git arms:
  - the round trip;
  - no filter registered (the refresh stages nothing);
  - a clean command that cannot run (`git add` refuses);
  - a worktree of a commit that carries the pointer is clean with the smudge, and its creation dies without it.
- [ ] The package version is bumped in the PR.

## Out of Scope

- The Engine's adoption (dropping `agentsMdPointer.mjs` and the two functions for the bin): an Engine leaf after this releases.
- `devindex`'s adoption: neomjs/devindex#135.
- The Brain's import of the marker: a follow-up to neomjs/neo-agent-brain#1606 once its skills pin carries it.

## Avoided Traps

- **The bin by name as the clean command.** Git does not see `node_modules/.bin`, so `neo-agent-skills-agents-md --clean` would not resolve from a filter. The command names the module path, as the Engine's names its own.
- **A smudge that generates the rules.** A checkout would then run node against a tree that is mid-checkout. `cat` writes the stored pointer, and the stage writes the rules.

## Related

#335 (the source), neomjs/neo#20749, neomjs/neo#20776 / neomjs/neo#20777, neomjs/devindex#135, neomjs/neo-agent-brain#1606.

Decision Record impact: none.

Live latest-open + closed sweep: latest 20 open (33 open, orange; folded under #335, the owning ticket, as its sub) + closed search ("AGENTS.md pointer filter": #12, #99, neither on this surface) at 2026-11-11T14:47Z; no equivalent. A2A in-flight claim sweep (last 30 at 14:48Z): no claim on this scope. MC sweep ("AGENTS.md pointer mechanism second consumer devindex copy skills package"): no prior ownership decision; #105 regenerated `devindex/AGENTS.md` by hand once. Own-assignment sweep: #335 (the source), #289 and #258 (unrelated).

Ownership: self-assigned (Grace).

Origin Session ID: c18ee0e6-21e0-45c2-8458-595620c7908d

Retrieval Hint: "AGENTS.md pointer git filter mechanism skills package second consumer devindex clean smudge required refresh"

## Timeline

- 2026-11-11T14:48:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T14:48:50Z @neo-opus-grace added the `enhancement` label
- 2026-11-11T14:48:51Z @neo-opus-grace added the `ai` label
- 2026-11-11T14:48:51Z @neo-opus-grace added the `build` label
- 2026-11-11T14:49:00Z @neo-opus-grace added parent issue #335
- 2026-11-11T14:56:23Z @neo-opus-grace cross-referenced by PR #347
- 2026-11-11T15:30:10Z @neo-opus-grace referenced in commit `f94ff37` - "fix(agents-md): the git runner stays module-private, so no outside argument picks a subcommand (#346)

CodeQL's second-order command injection check flagged runGit: exported, it let any caller hand git arguments such as --upload-pack. Each caller in the module names its own subcommand, and the registration's injectable git seam is unchanged."
- 2026-11-11T15:49:19Z @neo-opus-grace referenced in commit `e6519d2` - "fix(agents-md): the README documents --pointer, and the real-git checks run in an isolated git environment (#346)

Under a git hook, GIT_DIR and GIT_INDEX_FILE sent the test checkouts' commits into the hook's repository, and a host config that signs commits or carries its own neo-agents-md filter broke them. The test now strips GIT_* and points GIT_CONFIG_GLOBAL at an empty file with GIT_CONFIG_NOSYSTEM, as test-commit-authorship.mjs does. The README documents the flag and the ./agents-md/pointer export."
- 2026-11-11T16:20:19Z @neo-opus-grace cross-referenced by #335
- 2026-11-11T17:32:14Z @tobiu referenced in commit `b3a73a9` - "feat(agents-md): the AGENTS.md pointer and its git filter ship beside the generator (#346) (#347)

* feat(agents-md): the AGENTS.md pointer and its git filter ship beside the generator (#346)

A consumer keeps a pointer in git and the generated rules in its working copy with one call: neo-agent-skills-agents-md --pointer registers the neo-agents-md filter (clean, a smudge, required), writes the rules and lets the index re-read the file. The module that held this in the Engine moves here, so neo and devindex call it instead of carrying copies, and the Brain can import the marker from neo-agent-skills/agents-md/pointer.

* fix(agents-md): the git runner stays module-private, so no outside argument picks a subcommand (#346)

CodeQL's second-order command injection check flagged runGit: exported, it let any caller hand git arguments such as --upload-pack. Each caller in the module names its own subcommand, and the registration's injectable git seam is unchanged.

* fix(agents-md): the README documents --pointer, and the real-git checks run in an isolated git environment (#346)

Under a git hook, GIT_DIR and GIT_INDEX_FILE sent the test checkouts' commits into the hook's repository, and a host config that signs commits or carries its own neo-agents-md filter broke them. The test now strips GIT_* and points GIT_CONFIG_GLOBAL at an empty file with GIT_CONFIG_NOSYSTEM, as test-commit-authorship.mjs does. The README documents the flag and the ./agents-md/pointer export."
- 2026-11-11T17:32:16Z @tobiu closed this issue
- 2026-11-11T17:37:03Z @neo-opus-grace cross-referenced by #20788
- 2026-11-11T18:18:35Z @shounakjoshi88-a11y referenced in commit `ac9a334` - "Merge remote-tracking branch 'origin/dev' into docs/critical-gates-drop-rule-count

dev has moved to 1.4.0 since this branch was rebased, and `skill-corpus`'s
Version bump step needs a release above dev's, so the version moves to
1.4.1 - which npm does not have yet.

dev also brought the AGENTS.md pointer and its git filter (the #346 merge),
whose generated corpus is in the tree now, so the corpus lint measures the
same tree the generator writes.

Signed-off-by: Shounak Joshi <shounakjoshi88@gmail.com>"
- 2026-11-11T18:18:55Z @shounakjoshi88-a11y cross-referenced by PR #338
- 2026-11-11T19:18:07Z @neo-gemini-maryam cross-referenced by PR #20791
- 2026-11-11T20:12:21Z @neo-opus-grace cross-referenced by #353

