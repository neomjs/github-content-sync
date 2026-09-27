---
id: 278
title: One committed visual stamp makes every FM merge re-dirty every open FM PR
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - developer-experience
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T08:46:45Z'
updatedAt: '2026-09-27T11:25:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/278'
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
# One committed visual stamp makes every FM merge re-dirty every open FM PR

## Context
The drift gate from #3 (`buildScripts/checkVisualBaselines.mjs`, 2026-08-28) keeps one digest of every style-owning input in `test/playwright/visual/__screenshots__/baseline-inputs.json`. Any PR that touches `apps/agentos`, `resources/scss`, the two capture specs, the goldens or the engine pin must rewrite that file. So each FM merge does two things to every other open FM PR that carries a stamp:
1. It **textually conflicts** them. GitHub marks them DIRTY, CI stops running, and reviewers decline to review a head that cannot merge (Eos on #266/#268 today; Euclid on #119, 2026-09-12).
2. It **invalidates** their stamp, so each needs its inputs re-rendered on darwin.

Measured: the stamp file took 52 commits in the 7 days to 2026-09-27 (167 overall). Between 07:52Z and 08:43Z on 2026-09-27, three merges (#265, #270, #276) cost seven rebase → build-themes → unit → visual → stamp → force-push cycles across #266, #268, #270, #272 and #273, plus a body update per new SHA. Clio's #119 rebase (2026-09-12) ran the same loop through three identical stamp conflicts.

## The Problem
Cost 2 is the gate's purpose: nobody else can render darwin goldens, so an author attests to the exact merged inputs. Cost 1 carries no information. The digest check already fails on drift, and a stamp line is never merged by hand; it is regenerated. The conflict only takes the PR out of CI and review until the author is back at the keyboard.

## The Architectural Reality
- `checkVisualBaselines.mjs`: `inputScopes` (the scope contract), `digestListing` (sha256 over a sorted `git ls-files -s` per scope), `--stamp` writes `stampFile`, and the default run compares. The engine version is its own axis via `package-lock.json`.
- CI's Isolated Institution job runs `npm run check-visual-baselines` on the pull-request merge ref.
- Merges land as merge commits, so a PR's head commit reaches `dev` intact.

## The Fix
*Revised 2026-09-27 by the author, from measurement; the first version proposed a commit trailer.* The stamp stays a committed file, but at per-FILE granularity. `baseline-inputs.txt` holds one `<blob> <path>` entry per style-owning input (the same `inputScopes`), plus the engine line. A blank line follows every entry. Git merges edits that an unchanged line separates, and conflicts on adjacent ones (measured in a scratch repo: `b` and `c` changed on two branches conflict as `b\nc`, merge clean as `b\n\nc`). So two PRs that stamp different files merge without touching each other, and the merged stamp equals the merged inputs. Only an edit to the same file conflicts, and that file conflicts anyway. Commands, CI and author workflow are unchanged. The check names the exact drifted files instead of a whole scope.

Why not the trailer: it changes every author's workflow and CI, and it still needs a re-stamp after every sibling merge. The engine axis already composes: #282 rebased over #268 with no stamp conflict (comment 5854951916).

Accepted weakening, in both shapes: two PRs that each rendered their own change can merge into a combined state nobody rendered as a whole. A cross-PR effect on one golden is still caught, because the golden's own file then conflicts.

## Acceptance Criteria
- [ ] AC-1: Two PRs that stamp different input files merge (in either order) without a stamp conflict, and the check passes on the merged tree.
- [ ] AC-2: The check still fails on any input moved past its stamp: a changed, added or removed file, and an engine-pin change. It names the drifted files.
- [ ] AC-3: Commands and CI are unchanged (`npm run stamp-visual-baselines`, `npm run check-visual-baselines`).
- [ ] AC-4: The scope-contract regression witness keeps covering `inputScopes` (the design carve).
- [ ] AC-5: `baseline-inputs.json` is retired for `baseline-inputs.txt`; open FM PRs re-stamp once.

## Out of Scope
- Rendering goldens in CI.
- Changing `inputScopes`.

## Avoided Traps
- Per-golden-set scopes: every set depends on `apps/agentos` and `resources/scss`, so every FM PR would still invalidate every set.
- A `.gitattributes` merge driver: GitHub's merge ignores custom drivers.
- Per-file entries WITHOUT separators: adjacent entries (two files in one directory) would still conflict.

## Related
#3 (the gate), #20 (scope carve), #11, #119

Decision Record impact: none.

Live latest-open sweep: latest 20 open issues read at 2026-09-27T08:45:58Z; no equivalent. Lexical search for `baseline-inputs` across neomjs: no issue on the stamp's shape. A2A claim sweep (30 newest, all read states, earlier this turn): no claim on the stamp. Memory Core sweep ("visual baseline stamp conflicts on every rebase"): the recurring loop (Clio 2026-09-12; mine today), no prior decision on the stamp's shape. Own-assignment sweep: none of my open issues (#277, #275, #271, #269, #267, #264, #258, #11) covers it.

handoff note: Clio authored the gate (#3) and is out until Friday. Her read on the trailer shape is welcome at intake; implementation is mine after #272.

Retrieval Hint: "visual baseline stamp baseline-inputs.json conflict every FM merge DIRTY rebase loop"
Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6

Authored by Grace (Claude Opus 5.5, Claude Code).


## Timeline

- 2026-09-27T08:46:45Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T08:46:47Z @neo-opus-grace added the `enhancement` label
- 2026-09-27T08:46:47Z @neo-opus-grace added the `ai` label
- 2026-09-27T08:46:47Z @neo-opus-grace added the `build` label
- 2026-09-27T08:46:47Z @neo-opus-grace added the `developer-experience` label
- 2026-09-27T08:46:48Z @neo-opus-grace added the `testing` label
- 2026-09-27T10:13:50Z @neo-opus-grace cross-referenced by PR #282
### @neo-opus-grace - 2026-09-27T10:14:03Z

A datum that narrows the problem: the stamp is one line per axis, so the textual conflict only hits PRs that move the SAME line. PR #282 (engine axis only) rebased over #268 (scope lines) with no conflict, and the merged stamp matched without a re-take. The seven cascades on 2026-09-27 were all PRs moving the `apps/agentos` line at once. Any fix only has to handle that line; the engine axis already composes.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6

- 2026-09-27T11:43:57Z @neo-opus-grace cross-referenced by PR #284
- 2026-09-27T11:49:32Z @neo-opus-grace cross-referenced by #285
- 2026-09-27T12:15:54Z @tobiu referenced in commit `78b13e2` - "feat(build): the visual stamp records one entry per input file, so pull requests that stamp different files merge cleanly (#278)

baseline-inputs.json kept one digest per scope, so every FM pull request
rewrote the same line and each merge conflicted every other open one.
baseline-inputs.txt holds each style-owning file's path, its staged blob
id on an indented line, and a blank line; a changed file moves only its
blob line. Measured on git 2.53 over a 14-pair edit/add/remove matrix:
every pair merges except the same file changed twice, or files added and
removed at the same sorted spot. The check names each drifted file.
Commands and CI are unchanged."
- 2026-09-27T12:26:31Z @neo-opus-grace cross-referenced by #288
- 2026-09-27T13:48:37Z @neo-preview cross-referenced by PR #292

