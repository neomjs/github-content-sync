---
id: 375
title: 'Release notes: stale precedent paths and no previous-note section check'
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - model-experience
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-13T14:02:41Z'
updatedAt: '2026-11-13T16:00:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/375'
author: neo-opus-vega
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
closedAt: '2026-11-13T16:00:49Z'
---
# Release notes: stale precedent paths and no previous-note section check

## Context

`release-notes/references/release-notes-workflow.md` is what an author loads to write the engine's next release note. Read against the engine at `dev` `08dfefc634` on 2026-11-13, three of its statements no longer match what the release tooling does, and its cut-readiness checklist misses the one comparison that would have caught the defect of the 13.4.0 note. The next note is written for the cut of 2026-11-23.

## The Problem

1. **§5 sends the author to files that are not there.** It lists the precedent set (`v13.0.0.md`, `v12.1.0.md`, `v11.24.0.md`, `v11.23.0.md`, `v11.22.0.md`) as "All in the engine's `.github/RELEASE_NOTES/`". That folder holds `v13.4.0.md` and nothing else.
2. **§6 item 4 says the file is kept.** "publish.mjs keeps the file: the authored note is the archive, and nothing re-materializes it." The note survives its own release and is removed by the next one's `prepare`.
3. **§6 item 6 calls shipped behavior a known observation.** "Known observation (epic-tracked): … a committed staging note surfaces its version in `releases.json` when the docs index regenerates pre-cut." The index now lists an unreleased note on purpose, without a date.
4. **§7 never compares the note with its predecessor.** The 13.3.0 note carried a `**Try it:**` line in its two-minute section and a `## ▶️ What you can try` section before Upgrading. The 13.4.0 staging note passed every §7 line without either. The operator saw it on release day, and it was repaired on the release branch by `81ca26e2e6` (neomjs/neo#20657).

## The Architectural Reality

- `Design authority:` `buildScripts/release/publish.mjs`, JSDoc of `dropReleasedNotes`: "Removes every note older than the version being released from `.github/RELEASE_NOTES`, so the release commit carries only the notes not yet released." Introduced with the 13.3.1 cut-prep (neomjs/neo#20023).
- `Design authority:` `buildScripts/docs/index/release.mjs`, module JSDoc: "A note without `publishedAt` is unreleased and gets no date." (neomjs/neo#20666)
- Released notes live in the corpus: `neomjs/github-content-sync`, `neo/release-notes/chunk-N/`. All five precedent files and `v13.3.0.md` are there (corpus head `d1fa5aa7`, 2026-11-13). The GitHub release body is the second copy.
- §5 already says a staging note "renders at `#/news/releases/<id>` once a local `releases.json` lists it". That route is where a section comparison and a render read happen in one look.
- The payload is loaded only when the skill fires; it is not turn-loaded.

## The Fix

In `release-notes/references/release-notes-workflow.md` only:

1. §5: name where the precedent set is read (the corpus path, or `gh release view <tag> -R neomjs/neo`), in place of the engine folder.
2. §6 item 4: say when the authored note leaves the engine (the next `prepare`) and where it then lives.
3. §6 item 6: state the index's behavior for an unreleased note as it is, without "known observation".
4. §7: one line. The final note is read once at its Portal route beside the previous release's note, and its section set is the previous one's; a section that is dropped is a decision recorded on the notes epic.

Replace the stale sentences; the file should not grow by more than the one §7 line.

## Decision Record impact

`aligned-with` ADR 0007 (compaction taxonomy): a correction inside a skill-loaded payload, no turn-loaded change.

## Acceptance Criteria

- [ ] AC-1: §5 names a location where all five precedent files resolve today.
- [ ] AC-2: §6 items 4 and 6 match `dropReleasedNotes` and the release index as quoted above.
- [ ] AC-3: §7 carries the comparison with the previous note's section set, at the Portal route.
- [ ] AC-4: the corpus lint (`npm run lint`) passes and the payload's byte size is stated before and after in the PR.

## Out of Scope

- A script that compares two notes' headings. One checklist line first; a second miss earns the script.
- The release tooling itself.
- A render check for images and diagrams beyond the one read at the Portal route.

## Avoided Traps

- Restoring old notes into the engine folder so §5 becomes true: `dropReleasedNotes` removes them at the next cut.
- Fixing the section set as a list in the skill: sections change by decision, and the previous note is the reference that stays current.

## Related

neomjs/neo#20023 · neomjs/neo#20657 · neomjs/neo#20666 · earlier corrections of this payload: #104, #124, #171, #327.

Live latest-open + closed sweep: the latest 20 open of 35 (orange band; no open ticket owns the release-notes payload) and closed searches ("release notes section", "release-notes precedent", "release notes checklist cut-readiness", "RELEASE_NOTES folder") at 2026-11-13T14:01Z. No equivalent; #104, #124, #171 and #327 corrected other lines of the same file. A2A sweep (30 rows, all read states): no claim on this surface. Memory Core: two queries on the 13.4.0 note's missing section; the lesson was recorded on 2026-11-09 and never filed. Own assignments: #5, #6, #302, none on this payload. Structure map: N/A, one existing payload file changes and none is placed.

Ownership: self-assigned; I fix it before the next note is written. Neither `good first issue` nor `help wanted`: already owned.

Origin Session ID: af919a58-840f-4b86-b199-c0bbed744b76
Retrieval Hint: "release notes skill fix: section set of the previous note, What you can try, dropReleasedNotes, precedent set location"

## Timeline

- 2026-11-13T14:02:41Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-13T14:02:43Z @neo-opus-vega added the `bug` label
- 2026-11-13T14:02:44Z @neo-opus-vega added the `documentation` label
- 2026-11-13T14:02:44Z @neo-opus-vega added the `ai` label
- 2026-11-13T14:02:44Z @neo-opus-vega added the `model-experience` label
- 2026-11-13T14:02:58Z @github-actions added the `needs-design-authority` label
### @neo-opus-vega - 2026-11-13T14:07:15Z

**Contract Ledger** (the skill payload is an agent-consumed surface; filed before the PR opens).

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `release-notes-workflow.md` §5: where the precedent set is read | The corpus layout (`neomjs/github-content-sync`, `neo/release-notes/chunk-N/`); `publish.mjs` `dropReleasedNotes` | Names the corpus, and says the engine keeps only unreleased notes | A release the corpus has not synced yet: its GitHub release body | The payload | All five precedent files resolve at corpus head `d1fa5aa7` |
| §6 item 4: the authored note's lifetime | `publish.mjs`, JSDoc of `dropReleasedNotes`, called from `prepare` | Kept through its own release, removed by the next `prepare` | None | The payload | The call site inside `prepare` |
| §6 item 6: an unreleased note in the release index | `buildScripts/docs/index/release.mjs`, module JSDoc | Listed in `releases.json` without a date, readable at its Portal route before the cut | None | The payload | The quoted JSDoc sentence |
| §7: the cut-readiness checklist | This ticket; the 13.4.0 specimen (`81ca26e2e6`) | One line added: read at the Portal route beside the previous release's note, sections kept or the drop recorded on the notes epic | The first note of a new release line has no predecessor; the line then covers the render read only | The payload | The checklist line itself |

— Vega (Fable 5.2, Claude Code) 🌿

- 2026-11-13T14:07:57Z @neo-opus-vega cross-referenced by PR #376
- 2026-11-13T14:09:59Z @neo-opus-vega referenced in commit `d1273d7` - "chore(release): 1.8.1 carries the release-notes correction (#375)

A maintainer pull request in this repository carries a new release, and a
docs change is a patch."
- 2026-11-13T14:13:17Z @neo-opus-vega referenced in commit `8641e3b` - "docs(release-notes): trim the correction to the corpus growth budget (#375)

The corpus lint admits 250 bytes of growth without a stated reason and the
first commit added 277. Two lines lose 30 bytes; nothing they say changes."
- 2026-11-13T15:43:45Z @neo-opus-vega referenced in commit `4e0bb1d` - "docs(release-notes): drop the clause that called the engine's notes unreleased (#375)

The current released note stays in the engine until the next prepare removes
it, as item 4 of the publish section says. The corpus pointer is unchanged."
- 2026-11-13T16:00:49Z @tobiu referenced in commit `f112ab9` - "docs(release-notes): name where released notes live and compare a note with its predecessor (#375) (#376)

* docs(release-notes): name where released notes live and compare a note with its predecessor (#375)

Three statements of the release-notes workflow no longer matched the release
tooling: the precedent set is read in the corpus, the authored note is removed
by the next prepare, and the release index lists an unreleased note without a
date on purpose. The cut-readiness checklist gains the comparison with the
previous release's note, which the 13.4.0 note would have failed.

* chore(release): 1.8.1 carries the release-notes correction (#375)

A maintainer pull request in this repository carries a new release, and a
docs change is a patch.

* docs(release-notes): trim the correction to the corpus growth budget (#375)

The corpus lint admits 250 bytes of growth without a stated reason and the
first commit added 277. Two lines lose 30 bytes; nothing they say changes.

* docs(release-notes): drop the clause that called the engine's notes unreleased (#375)

The current released note stays in the engine until the next prepare removes
it, as item 4 of the publish section says. The corpus pointer is unchanged."
- 2026-11-13T16:00:51Z @tobiu closed this issue

