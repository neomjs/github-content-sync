---
id: 725
title: 'The receiver spec publishes in place, so its startup arm can read a torn manifest'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T19:31:33Z'
updatedAt: '2026-10-01T20:05:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/725'
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
closedAt: '2026-10-01T20:04:02Z'
---
# The receiver spec publishes in place, so its startup arm can read a torn manifest

## Context

Item 4 of Clio's wake/plane queue (A2A, 2026-10-01 19:27Z), from Ada's defect-note at 19:16Z. `receiver.spec`'s arm "a publish during startup is still adopted — the served revision is the one that was LOADED" failed once in a full local unit run (PR #724's head), then passed 5/5 alone. It is the same race family as #717, which #718 fixed, but a different arm. Like #717, it surfaces as a head-suite red that #650's comparator counts as introduced.

## The Problem

Reproduced: running only that arm 400 times at 12 workers gives **2 failures**, each `SyntaxError: Unexpected end of JSON input` from `startWakeReceiver` → `loadManifestWithRevision` → `loadWakeReceiverManifest` (`receiver.mjs:89`).

The spec's shared `write` helper publishes **in place** (`fs.writeFile(manifestPath, …)`, `receiver.spec.mjs:287-290`). The arm publishes while the receiver is starting, so boot can read the file between truncate and write. Boot retries only on a revision mismatch, not on a parse error, so the half-written file rejects the start.

## The Architectural Reality

- **The real publisher never exposes a half-written file.** `buildReceiverManifest` writes a staging file in the same directory and renames it into place (`buildReceiverManifest.mjs:462-485`). The receiver's revision (`mtime:size:ino`) is built on that contract: a rename gives every publish a new inode.
- **The in-place helper exercises a publish shape production never produces,** and this arm is the only one that publishes during boot, where a read error has no retry.
- Arms that write malformed content in place on purpose (`:380`, `:547`) test refusal, not publishing, and stay as they are.

## The Fix

`write` publishes as the real publisher does: a uniquely named staging file in the same directory, with the requested mode, renamed into place. Spec-only. `receiver.mjs` is unchanged.

## Acceptance Criteria

- [x] AC-1: the startup arm runs 1,200 times at 12 workers with no failure, where `dev`'s spec fails at that rate (2 in 400 measured). *Delivered: 0 in 1,200 on PR #726, against 2 in 1,200 on `dev`'s spec under the same load. Merged as `e6f0bb0`.*
- [x] AC-2: every `receiver.spec` arm passes, repeated 20 times. The 0644 mode refusal still refuses. *Delivered: 622/622 (`--repeat-each 20`), PR #726.*

## Out of Scope

Boot tolerating an in-place writer (retrying a parse error when the revision moved during the read). Production publishes by rename, and an in-place writer would also break the inode identity #717 depends on, so tolerating half of that contract would mislead.

## Related

#717 / PR #718 (the same-mtime arm) · #650 (the comparator these reds trip) · #720

## Sweeps

- Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T19:31:05Z. No equivalent. #717 is closed (its PR #718 fixed the other arm).
- Exact: `gh search issues --owner neomjs` for "receiver.spec startup", "publish during startup" and "Unexpected end of JSON input receiver": none.
- MC sweep: "receiver spec publish during startup still adopted fails under full suite; manifest half-written read at boot", 6 results. They are older wake-route incidents (envelope, manifest snapshot); none on this arm.
- A2A: the 15 newest, all read-states, to 19:30:58Z. My own item-4 claim (19:28:20Z) is the only claim, and Clio's settled plan gives it to the first claimant.
- Own-assignment: #684, #710, #712 and older lanes; none overlapping.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "receiver.spec publish during startup torn read Unexpected end of JSON input in-place write helper rename"

🖖 Grace (Claude Opus 5.5, Claude Code)


## Timeline

- 2026-10-01T19:31:35Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T19:31:35Z @neo-opus-grace added the `bug` label
- 2026-10-01T19:31:35Z @neo-opus-grace added the `ai` label
- 2026-10-01T19:31:35Z @neo-opus-grace added the `testing` label
- 2026-10-01T19:35:45Z @neo-opus-grace cross-referenced by PR #726
- 2026-10-01T20:04:02Z @tobiu referenced in commit `e6f0bb0` - "test(wake): the receiver spec publishes by rename, so its startup arm never reads a torn manifest (#725) (#726)

The shared write helper wrote the manifest in place; the startup arm publishes while boot reads, and a half-written file failed the start (2 in 400 under load). buildReceiverManifest stages and renames, so the spec now does too."
- 2026-10-01T20:04:02Z @tobiu closed this issue

