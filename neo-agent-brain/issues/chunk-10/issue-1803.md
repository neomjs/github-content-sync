---
id: 1803
title: A fresh Codex Start asks for wake-hook approval when only the Fleet package changes
state: CLOSED
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-16T13:35:17Z'
updatedAt: '2026-11-16T16:07:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1803'
author: neo-gpt-emmy
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
blocking:
  - '[ ] 1005 Package the Codex wake hook independently of unrelated Fleet changes'
closedAt: '2026-11-16T16:07:18Z'
---
# A fresh Codex Start asks for wake-hook approval when only the Fleet package changes

## Context

An installed seat's previous approved hook continued to run after a Fleet update. A fresh managed Start on 16 November selected the new sealed generation. The two generation-qualified paths in its wake hook changed and Codex displayed **Modified**. An ordinary submit subsequently invoked the new definition in the new process. The old configuration digest can be reproduced by replacing those two path components alone. The entrypoint bytes matched, **but the nonce consumer actually changed**; this particular update cannot serve as an unchanged-code control. Prior installed approval is recorded on [the native wake trial](https://github.com/neomjs/neo-agent-brain/issues/1735#issuecomment-6893960428).

## The Problem

Every new Fleet package can have a different immutable location even if all the code that handles a submitted prompt stays identical. On a fresh Start the command embeds those locations. The operator has to inspect the same capability again and has no way to tell a package-only change from an admission-code change. [Codex reviews hook definitions](https://developers.openai.com/codex/hooks#review-and-trust-hooks).

## The Architectural Reality

Design authority: the earlier [surviving-seat repair](https://github.com/neomjs/neo-agent-brain/issues/1750) intentionally selects the new runtime on a fresh Start, following [ADR 0034 §2.5](https://github.com/neomjs/neo-agent-institution/blob/dev/learn/agentos/decisions/0034-electron-shell-architecture.md). The product request now explicitly includes fresh Starts. The host packages executable code, Brain prepares the hook and verifies the exact trusted vendor inventory plus an ordinary invocation from the current process. The current entrypoint imports its nonce consumer dynamically, which in turn has native and third-party dependencies. A digest of the entry file alone is insufficient. A prior preparation receipt does not protect a whole generation from collection.

## The Fix

Decouple the identity of the executable hook from the identity of the complete Fleet package while binding each Start to the verified hook it declared. Demonstrate a location-independent immutable payload with the existing nonce consumer on two disposable package generations. Decide its delivery and retention against the host's existing sealed runtime owner before adopting it. Never copy or synthesize the vendor's trust decision. A mutable stable path that silently follows new code is not a solution.

## Acceptance Criteria

- [ ] Two fresh Starts with different package IDs but the same complete executable hook keep the same vendor hook definition and hash. Verify against genuine Codex inventory in an isolated home, and separately exercise the signed single-delivery and duplicate controls.
- [ ] A real executable, hook, dependency, or security-protocol change selects a new definition requiring native review; tamper, foreign binding, unproved closure or missing custody fails closed without sending a pending wake. Preserve legacy previously enrolled seats through a defined upgrade.
- [ ] Preparation, receiver verification, collision-safe publication and artifact lifetime agree. Repeated Start, rollback, and collection cannot retarget an approved definition underneath a running or enrolled seat. Account for package/storage cost and the normal installer, not only a standalone fixture.
- [ ] **Installed validation after the source change:** on one enrolled seat, record two managed Starts across a verified package-only update with one genuine operator grant and a signed model-side receipt. Installed owner: #571; native delivery owner: #1735. No production credential, seat configuration, or private hook input belongs in the receipt.

Origin Session ID: 289da399-da26-4b13-ad71-dae9311704bf

## Timeline

- 2026-11-16T13:35:19Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-16T13:35:19Z @neo-gpt-emmy added the `ai` label
- 2026-11-16T13:35:19Z @neo-gpt-emmy added the `security` label
- 2026-11-16T13:35:20Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-16T13:43:24Z @neo-gpt-emmy cross-referenced by #1005
- 2026-11-16T13:43:34Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-16T14:05:17Z @neo-gpt-emmy cross-referenced by PR #1805
- 2026-11-16T14:09:24Z @neo-gpt-emmy referenced in commit `82afb19` - "fix(fleet): verify physical hook artifact entries under Electron (#1803)"
- 2026-11-16T14:14:03Z @neo-gpt-emmy referenced in commit `d54f292` - "fix(test): supply the Start verifier to the Fleet pump fixture (#1803)"
- 2026-11-16T14:18:13Z @neo-gpt-emmy referenced in commit `9194c0a` - "fix(fleet): refuse unretained hooks before first enrollment (#1803)"
- 2026-11-16T14:23:50Z @neo-gpt-emmy referenced in commit `6ac4be9` - "fix(fleet): keep Codex hook approval across package-only Starts (#1803)"
- 2026-11-16T14:23:50Z @neo-gpt-emmy referenced in commit `26e3996` - "fix(fleet): verify physical hook artifact entries under Electron (#1803)"
- 2026-11-16T14:23:50Z @neo-gpt-emmy referenced in commit `71432b7` - "fix(test): supply the Start verifier to the Fleet pump fixture (#1803)"
- 2026-11-16T14:23:51Z @neo-gpt-emmy referenced in commit `7bf63ea` - "fix(fleet): refuse unretained hooks before first enrollment (#1803)"
- 2026-11-16T14:25:22Z @neo-opus-ada cross-referenced by #1007
### @neo-gpt-emmy - 2026-11-16T14:38:01Z

**Normal macOS package receipt, 16 November.** Institution `ceabcb7dfab435322a98ff31ab99e76c1ea48698`, Brain `7bf63ea6a935f93101912edd02960471eb567b48`, Engine `e9661d43ff13e91e40c9eaddc85225333a906ac0`, Electron 43.8.1. I installed the checked-in product lockfile and ran the normal unsigned arm64 distribution build with a separate output directory. The ZIP is 346,920,843 bytes, SHA-256 `7e6de8dd342dc73058f4a6175a2f32065c8a9aff59f298158ecc4d342d1d5583`.

I checked every archive entry's CRC, read back all three owner revisions, and verified that the packaged v1 declaration’s 21 first-party modules and eight npm package roots are present. The hook, reader, Fleet manager and open-work selector match the reviewed Brain tree byte-for-byte. The packaged shell’s ASAR contains the matching publication, runtime, maintenance and main-process implementations.

The separate disposable vendor inventory preserved the same hook/config hash across a package-only change; changing the nonce consumer changed the hash. The HMAC fixture delivered once and denied replay. The archive has not replaced the installed app. The next installed step on #571 is one genuine grant for the new address, then a planned package-only update and fresh Start on the same seat, followed by a signed model-side receipt. The separately signed release carrier is tracked in neomjs/neo-agent-institution#1007.

[Brain reader](https://github.com/neomjs/neo-agent-brain/pull/1805) · [Institution packager](https://github.com/neomjs/neo-agent-institution/pull/1006)

- 2026-11-16T14:41:43Z @neo-gpt-emmy marked this issue as blocking #1005
- 2026-11-16T16:07:18Z @tobiu referenced in commit `08fba72` - "fix(fleet): keep Codex hook approval across package-only Starts (#1803) (#1805)

* fix(fleet): keep Codex hook approval across package-only Starts (#1803)

* fix(fleet): verify physical hook artifact entries under Electron (#1803)

* fix(test): supply the Start verifier to the Fleet pump fixture (#1803)

* fix(fleet): refuse unretained hooks before first enrollment (#1803)"
- 2026-11-16T16:07:19Z @tobiu closed this issue

