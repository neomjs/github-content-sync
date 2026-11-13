---
id: 1660
title: Antigravity and Codex acquire through the shared archive pipeline
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - refactoring
  - agent-os
assignees: []
createdAt: '2026-11-13T02:12:24Z'
updatedAt: '2026-11-13T05:23:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1660'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1531
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1658 The Fleet keeps Claude Desktop seats on the newest admitted build'
blocking: []
---
# Antigravity and Codex acquire through the shared archive pipeline

## Context

#1658 adds `ai/services/fleet/acquireHarnessArchive.mjs`, the half of a harness acquisition that does not depend on the vendor: runs for one family never overlap, a build the same admitting code rejected is not fetched again, the archive streams to staging under a byte cap while its digest is computed, nothing is extracted unless the digest matches, `ditto` extracts it, and the one `.app` goes to `materializeHarnessGeneration`. Claude Desktop's source (`claudeDesktopUpdate.mjs`) is its first caller. The two older sources still carry their own copies of that half.

## The Problem

`antigravityUpdate.mjs` (711 lines) and `codexDesktopUpdate.mjs` (709 lines) each re-implement it: a `refusal` helper, `defaultAdmitterId`, the `acquireChains` / `inProcessRejections` maps, a streaming download (`streamAndVerifyAntigravityArchive`; Codex's `defaultDownloadArchive`), `ditto` extraction, the single-`.app` lookup, and `compareDottedVersions` (exported by Antigravity, private in Codex). With three copies of one pipeline, a fix to the cap, the staging sweep or the rejection memo lands in one and silently not in the others.

## The Architectural Reality

- `acquireHarnessArchive({root, family, version, archiveUrl, algorithm, expectedDigest, expectedSize, admitterId, ...seams})` checks the digest (and the size, when the source names one) before extraction, records the archive URL as the generation's `source`, and always removes its staging. Callers run it inside `serializeHarnessAcquisition({root, family}, run)`.
- Antigravity's `latest-<arch>-mac.yml` names a base64 SHA-512 and the size, so it maps directly: `algorithm: 'sha512'`, `expectedDigest: Buffer.from(sha512, 'base64')`, `expectedSize`.
- Codex's Sparkle item names a length and an Ed25519 signature over the archive (`verifySparkleEdSignature`), not a digest. The pipeline needs one optional pre-extraction hook for that, e.g. `verifyArchive({archivePath, byteLength})`, called where the digest check runs today.
- Once #1661 merges, `acquireHarnessArchive` hands `version` to admission as `expectedShortVersion`, and admission refuses a bundle whose signed `CFBundleShortVersionString` differs (`bundle-version-mismatch`) before publication. Each caller passes its feed's short version: Antigravity's manifest `version`, Codex's `candidate.shortVersion` (`sparkle:shortVersionString`), never `sparkle:version`. Both `alreadyAdmitted` checks already assume these equal the bundle's; Codex's `title` fallback may not.

## The Fix

1. Give `acquireHarnessArchive` the optional `verifyArchive` hook (runs after the download and before extraction; a throw refuses like a digest mismatch). Each run checks a digest, the hook, or both.
2. `acquireAntigravityUpdate` keeps its manifest reading and calls `acquireHarnessArchive` with SHA-512 and the size. Delete its own streaming verifier, extraction, `.app` lookup, run chain, rejection memo, admitter fingerprint and version comparison; import them instead.
3. `acquireCodexDesktopUpdate` does the same, with its Ed25519 check passed as `verifyArchive`.

## Acceptance Criteria

- [ ] Neither `antigravityUpdate.mjs` nor `codexDesktopUpdate.mjs` defines its own streaming download, extraction, `.app` lookup, run chain, rejection memo, admitter fingerprint or `compareDottedVersions`; both import them from `acquireHarnessArchive.mjs`.
- [ ] `acquireHarnessArchive` runs a `verifyArchive` hook before extraction, and a hook that throws extracts and admits nothing (unit, in `acquireHarnessArchive.spec.mjs`).
- [ ] `antigravityUpdate.spec.mjs`, `codexDesktopUpdate.spec.mjs` and `acquireHarnessArchive.spec.mjs` pass. A refusal code may change to the shared one (for example `sha512-mismatch` to `digest-mismatch`) where the spec asserts it; no behavior is dropped.
- [ ] For each family, a spec through the real `materializeHarnessGeneration` shows the version it passes is the bundle's short version: a matching feed is admitted, a mismatching one is refused (`bundle-version-mismatch`) and the selection stays.
- [ ] The two modules lose more lines than the pipeline gains (`wc -l` before and after in the PR).

## Out of Scope

- Changing either vendor's feed handling, signature rules or what admission checks, beyond the version binding the shared pipeline brings (#1661).
- The Codex opt-in (its own leaf under #1531).

## Avoided Traps

- **A fourth copy.** A new family source calls the pipeline; it never copies a sibling.
- **Folding the vendor parts in.** Manifest, appcast and cask parsing stay with their families; only the vendor-independent half is shared.

## Related

#1531 (parent) · #1658 (the pipeline and its first caller) · #1537 (Codex acquisition) · #1539 (Antigravity acquisition)

Decision Record impact: none.

Live latest-open + closed sweep: latest open, plus a search for `antigravityUpdate codexDesktopUpdate duplicate OR shared acquisition` across open and closed, at 2026-11-13T02:11Z; no equivalent found. Backlog band: 155 open (red), so this folds under #1531. A2A claim sweep: latest 30 messages, all read states; no claim on harness acquisition. MC sweep: "harness acquisition modules each re-implement download, digest check, extraction and rejection memo", 5 results, no prior decision. Own-assignment sweep: #1531 is the parent; #571, #133 and #1508 do not cover it.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The unit specs run without macOS: every host effect has a seam.

Origin Session ID: 776d12fd-202a-486b-88c0-41605d050556

Retrieval Hint: "one archive pipeline for every harness acquisition; Antigravity and Codex keep their own copies of download, verify and extract"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T02:12:26Z @neo-opus-ada added the `enhancement` label
- 2026-11-13T02:12:26Z @neo-opus-ada added the `good first issue` label
- 2026-11-13T02:12:26Z @neo-opus-ada added the `help wanted` label
- 2026-11-13T02:12:27Z @neo-opus-ada added the `ai` label
- 2026-11-13T02:12:27Z @neo-opus-ada added the `refactoring` label
- 2026-11-13T02:12:27Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T02:12:30Z @neo-opus-ada added parent issue #1531
- 2026-11-13T02:12:32Z @neo-opus-ada marked this issue as being blocked by #1658
- 2026-11-13T02:12:49Z @neo-opus-ada cross-referenced by #1658
- 2026-11-13T02:14:13Z @neo-opus-ada cross-referenced by PR #1661
- 2026-11-13T02:16:01Z @neo-opus-ada cross-referenced by #1662
- 2026-11-13T02:44:06Z @neo-opus-ada cross-referenced by #1531

