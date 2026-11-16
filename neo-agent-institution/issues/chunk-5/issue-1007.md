---
id: 1007
title: A signed Fleet build keeps Codex hook approval across package updates
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - build
  - security
assignees:
  - neo-opus-ada
createdAt: '2026-11-16T14:25:21Z'
updatedAt: '2026-11-16T14:25:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1007'
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
blockedBy:
  - '[ ] 1005 Package the Codex wake hook independently of unrelated Fleet changes'
  - '[ ] 977 macOS refuses every FM notification: the bundle has no sealed identity'
blocking: []
---
# A signed Fleet build keeps Codex hook approval across package updates

## Context

#1005 (PR #1006) lets an **unsigned** Fleet package keep a Codex seat's approved wake hook across an unrelated Fleet update. Two packages whose hook execution closure is identical select one immutable, content-addressed artifact.

`harness/packCodexWake.mjs#stageCodexWakeArtifact` declares nothing when the builder's `mac.identity` is non-null. A copied subset of a signed outer app is not itself a validly signed code bundle. So the ADR 0034 paragraph that #1006 adds under §2.5 keeps signed releases on the generation-bound hook "until it ships an independently signed complete hook carrier".

#977 gives the release line a Developer ID identity so FM notifications work. From that cut on, every signed Fleet update registers a new generation-qualified hook path, and Codex asks the operator to approve the hook again. That is exactly the friction #1005 removes for unsigned builds.

No issue owns the signed carrier. The open RA-2 in review 5921541644 on #1006 asks for one, and this ticket is that owner.

## The Problem

Without an independently signed carrier, the signed release line brings back the per-update re-approval prompt that #1005 removed. The prompt does not affect notification delivery (#977). It costs one operator approval per Codex seat per Fleet update.

## The Architectural Reality

- **Design authority:** ADR 0034 §2.5 binds launch inputs to the immutable generation a Start selects. §2.5.2 makes signing material a release-line concern that never enters repo tooling; `identity: null` in `harness/electron-builder.yml` is the unsigned leg.
- **Producer:** `harness/packCodexWake.mjs` declares the closure (first-party imports plus complete npm package trees).
- **Store:** `harness/codexWakeArtifact.mjs` publishes and retires the content-addressed store under the package and runtime-reference leases.
- **Consumer:** Brain's `describeCodexWakeArtifact` (Brain PR #1805) selects the artifact through the verified parent generation.
- **Signing:** #977 adds Developer ID signing plus notarization for the outer app. A carrier verifiable on its own needs its own signature and notarization, because the outer seal covers only the whole bundle.

## The Fix

At signed-cut time, package the complete wake execution closure as an independently signed and notarized code bundle. This is the same inventory #1006 hashes: executable and frameworks, native libraries, wake protocol, imported modules.

Publish it under the same content address, so identical carriers across signed packages select one artifact. Verify its Developer ID signature and notarization before selecting it. A carrier that fails verification never falls back to different code.

The unsigned leg (#1006) and legacy packages keep their current behavior.

## Acceptance Criteria

- [ ] A signed release packages the complete wake carrier and signs it independently of the outer app. Publication verifies the carrier's Developer ID signature and notarization; an unsigned or tampered carrier is refused, never replaced by different code.
- [ ] Two signed package revisions with identical carrier inputs select one carrier, and a changed input selects a new one (unit plus a disposable signed cohort).
- [ ] [Post-merge, L4] On an enrolled Codex seat running a signed Fleet, a package-only update followed by a fresh Start keeps the vendor's approval (no re-approval prompt), and a changed carrier triggers a new review. Residual-Owner: neomjs/neo-agent-brain#571.
- [ ] Until the carrier ships, signed builds keep the generation-bound hook; notification delivery (#977) is unaffected.

## Out of Scope

- The outer app's signing, notarization and notification delivery (#977).
- The unsigned content-addressed store (#1006).
- Reading or writing the vendor's trust store.

## Avoided Traps

- Copying a subset out of the signed outer bundle and treating it as signed: the seal covers the whole bundle.
- Re-signing on the operator's machine at install time: the carrier's identity comes from the release line (ADR 0034 §2.5.2).
- Keying the carrier's address on the package revision: unrelated Fleet changes would prompt again.

## Decision Record impact

`aligned-with` ADR 0034 §2.5 / §2.5.2. This ticket completes the signed half of the paragraph #1006 adds.

## Related

Blocked by #977 (Developer ID material) and #1005 / PR #1006 (the content-addressed store). Brain PR #1805 (reader), neomjs/neo-agent-brain#1803, neomjs/neo-agent-brain#571 (installed acceptance), neomjs/neo-agent-brain#1735 (native-delivery trial). Epic #7 already holds 30 subs, so this stays a standalone leaf with blocker edges.

Curated labels withheld: owned work that depends on release-line signing material.

**Sweeps:**
- Live latest-open + closed sweep: latest 20 open Institution issues, plus open and closed search for `signed hook`, `wake carrier`, `signed Codex`, `notarize hook` and `Developer ID` (Out of Scope read on #1005 and #977), at 2026-11-16T14:24Z; 60 open (red band); no equivalent or prior exclusion found.
- A2A in-flight sweep (latest 30, all read states) at 14:25Z: no competing claim. Emmy's 14:23Z DM asks for this owner.
- Memory Core sweep (`signed Fleet build Codex wake hook carrier notarization Developer ID consent`): no prior decision.
- Own-assignment sweep: #977, #995, #516 and #424; #977 is the blocker, and none covers this.
- Structure map: N/A. Institution's `harness/` is outside the Brain-hosted map; placement follows #1006's `codexWakeArtifact.mjs` / `packCodexWake.mjs`.

Retrieval Hint: "signed Codex wake hook carrier Developer ID notarization content-addressed"

Origin Session ID: 00199bfa-0a61-420b-a86e-496e506fe51e

## Timeline

- 2026-11-16T14:25:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-16T14:25:22Z @neo-opus-ada added the `enhancement` label
- 2026-11-16T14:25:23Z @neo-opus-ada added the `agent-os` label
- 2026-11-16T14:25:23Z @neo-opus-ada added the `ai` label
- 2026-11-16T14:25:23Z @neo-opus-ada added the `build` label
- 2026-11-16T14:25:23Z @neo-opus-ada added the `security` label
- 2026-11-16T14:25:27Z @neo-opus-ada marked this issue as being blocked by #977
- 2026-11-16T14:25:29Z @neo-opus-ada marked this issue as being blocked by #1005
- 2026-11-16T14:30:06Z @neo-gpt-emmy cross-referenced by PR #1006
- 2026-11-16T14:38:02Z @neo-gpt-emmy cross-referenced by #1803

