---
id: 104
title: release-notes §2 undercounts the window by 30% from a month-stale mirror
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-23T09:42:06Z'
updatedAt: '2026-09-23T11:54:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/104'
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
blocking: []
closedAt: '2026-09-23T11:54:08Z'
---
# release-notes §2 undercounts the window by 30% from a month-stale mirror

## Context

@tobiu, 2026-09-23, on neomjs/neo#19091: the engine's `resources/content` is a month stale, the live corpus is [`neomjs/github-content-sync` `neo/`](https://github.com/neomjs/github-content-sync/tree/dev/neo), and the release-notes skill needs the update.

Found while deriving the 13.2 window for neomjs/neo#14800.

## The Problem

`release-notes-workflow.md` §2 has two silent defects. Both are in 0.1.14, which is byte-identical to the copy installed on neo `dev`.

**1. The scope instrument reads a frozen mirror.**
- §2 prescribes `node buildScripts/release/analyzeClosedSinceRelease.mjs`. It reads `resources/content/{issues,pulls}` under the engine checkout (`analyzeClosedSinceRelease.mjs:30-32`).
- That mirror was last synced at `e7874db2d2` (2026-08-26). The pipeline has published to `neomjs/github-content-sync` since then.

Same script, same cutoff (`2026-07-03T21:40:13Z`), measured on 2026-09-23:

| Source | Merged PRs | Closed issues | Epics closed |
|---|---|---|---|
| engine `resources/content`, as §2 prescribes | 1,365 | 1,534 | 10 |
| corpus `neo/` at `267fedf` | **1,953** | **2,170** | **27** |

- **It exits 0.** Its only warning is the generic "local mirror only" freshness line.
- **The size of the miss:** an author following §2 undercounts the 13.2 window by 588 merged PRs (30 %) and 17 of 27 epic closures.
- **Live confirmation:** GitHub search finds 721 + 714 + 518 = 1,953 merged PRs for July, August and September, matching the corpus.

**2. The cutoff one-liner finds the wrong commit.**
- `git log -1 --format=%ai -S '"version"' package.json` returns `e836e307a0` (2019-11-12), the commit that added the key.
- `-S` counts occurrences, and a version bump doesn't change the count.
- `-G '"version"'` returns `615d76e7d2` (2026-07-03, "Release v13.1.0").
- The release tag is `13.1.0`, with no `v`. `gh release view 13.1.0 --json publishedAt` gives 2026-07-03T21:40:28Z.

## The Architectural Reality

- **The corpus keeps the engine's layout for each origin:** `neo/{issues,pulls,archive,discussions}`. It publishes on an hourly schedule; observed gaps run 2.9–6.0 h ([D#19051](https://github.com/neomjs/neo/discussions/19051)). Nothing has been archived since 13.1.0, so its `issues/` and `pulls/` hold the whole window.
- **The analyzer derives its root from its own location** (`neoRoot = ../..`) and takes no root argument. Today, pointing it at the corpus needs a *copy* of the script beside a data link: `resources/content/{issues,pulls}` → a sparse checkout of the corpus's `neo/{issues,pulls}`. That recipe is on neomjs/neo#14800 (comment 5792321883). A declared corpus root for the engine's readers is neomjs/neo#17416's work: cornerstone 3, with @neo-opus-vega's reader census.
- **A window can now span several release lines.** Since the repository split, one window can hold several lines. For the engine's 13.2, about 958 of the 1,953 merged PRs are Agent OS or Fleet Manager work: 855 by title scope (`ai`, `agentos`, `fleet`, `memory-core`, …), and 103 more by changed paths where the title names no scope. The analyzer's "PRs By Title Scope" and "Issues By Parent Epic" tables separate them: transferred parents show up as the other repositories' epic numbers.

## The Fix

In §2:

1. **Source:** name the corpus as the scope source. Run the analyzer over `neomjs/github-content-sync` `<repo>/`, through the copy-plus-data-link recipe until the engine script takes a declared root. Keep the live GitHub count check at the cut.
2. **Cutoff:** replace the one-liner with `-G '"version"'`, or with the release tag's `publishedAt`.
3. **Split-era windows:** add one line. Merged PRs and resolved tickets are the unit, not commits. Count the release line's own, and separate other lines by title scope and transferred parent epics.
4. **Version:** bump `package.json` in the same PR.

## Acceptance Criteria

- [ ] AC-1 §2 names the corpus as the scope source, and nothing in the skill tells an author to count from the engine's `resources/content`.
- [ ] AC-2 Following §2 literally on the 13.2 window reproduces the corpus counts (1,953 / 2,170 / 27 on 2026-09-23), not the mirror's.
- [ ] AC-3 The cutoff instruction resolves 13.1.0 to `615d76e7d2`, or to the tag's date.
- [ ] AC-4 The split-era line is present: PRs and tickets are the unit, and other lines are separated by scope and parent epic.
- [ ] AC-5 `package.json` is version-bumped, and the repository's own gates pass.

## Out of Scope

- **The engine script's root option.** That is neomjs/neo#17416's reader work.
- **The other skills that grep the engine mirror:** ticket-create, ticket-intake, ticket-triage, epic-review and the ideation Gate-0 sweep. D#19051's cutover AC 2 repoints them in one wave. They read the same month-stale mirror *today*, which may argue for moving that wave earlier; that is D#19051's call.
- **§6's staging path.** It moves with D#19051's R1, after 13.2.

## Related

neomjs/neo#14800 · neomjs/neo#19102 · neomjs/neo#19091 · neomjs/neo#17416 · [D#19051](https://github.com/neomjs/neo/discussions/19051)

---

Live latest-open sweep: the latest 20 open issues in this repository at 2026-09-23T09:40:23Z; no release-notes or mirror ticket exists. Keyword searches for "release-notes" and "resources/content" found only closed, unrelated tickets.
A2A claim sweep: the newest messages, all read-states; no claim on the release-notes skill.
MC sweep: "release-notes skill analyzeClosedSinceRelease stale engine mirror undercount corpus" returned no prior decision.
Own-assignment sweep: #100 and #76 in this repository; no overlap.

Origin Session ID: bf94c4a1-fded-4546-87d6-73df33928275
Retrieval Hint: "release-notes skill scope derivation stale mirror corpus github-content-sync cutoff -S -G"


## Timeline

- 2026-09-23T09:42:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-23T09:42:08Z @neo-opus-grace added the `bug` label
- 2026-09-23T09:42:08Z @neo-opus-grace added the `documentation` label
- 2026-09-23T09:42:08Z @neo-opus-grace added the `ai` label
- 2026-09-23T09:43:12Z @neo-opus-grace cross-referenced by #14800
- 2026-09-23T09:45:40Z @neo-opus-grace cross-referenced by PR #19091
- 2026-09-23T11:02:37Z @neo-opus-vega cross-referenced by #105
- 2026-09-23T11:03:28Z @neo-opus-vega cross-referenced by #17416
- 2026-09-23T11:29:06Z @neo-opus-grace cross-referenced by PR #106
- 2026-09-23T11:46:06Z @neo-opus-grace referenced in commit `c297f1c` - "fix(release-notes): §2's corpus recipe is a runnable command block (#104)

The analyzer resolves its root two levels above itself and imports
gray-matter, so the copy needs a named scratch root, the engine's
node_modules, and gh release view -R for the release line. The block,
run verbatim from an engine checkout, reproduces 1,953 / 2,170 / 27 at
corpus 267fedf.

[skill-growth-justified: a runnable temporary recipe for §2's corpus scope; it retires when neomjs/neo#17416 gives analyzeClosedSinceRelease.mjs a declared content root]"
- 2026-09-23T11:54:08Z @tobiu referenced in commit `fe1a943` - "Merge pull request #106 from neomjs/grace/104-release-notes-corpus

fix(release-notes): §2 derives scope from the corpus, not the frozen engine mirror (#104)"
- 2026-09-23T11:54:08Z @tobiu closed this issue
- 2026-09-23T12:03:33Z @neo-opus-vega cross-referenced by PR #107
- 2026-09-23T12:28:05Z @neo-opus-ada cross-referenced by #103
- 2026-09-23T12:50:53Z @neo-opus-grace cross-referenced by #19118

