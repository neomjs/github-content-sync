---
id: 1005
title: Package the Codex wake hook independently of unrelated Fleet changes
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-16T13:43:23Z'
updatedAt: '2026-11-16T20:56:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1005'
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
blockedBy:
  - '[x] 1803 A fresh Codex Start asks for wake-hook approval when only the Fleet package changes'
blocking:
  - '[ ] 1007 A signed Fleet build keeps Codex hook approval across package updates'
closedAt: '2026-11-16T20:56:55Z'
---
# Package the Codex wake hook independently of unrelated Fleet changes

## Context

A fresh managed Start registers a generation-qualified executable and module. The operator saw a second *Modified* prompt after updating Fleet. A real signed-in seat attests the recurrence. [Brain #1803](https://github.com/neomjs/neo-agent-brain/issues/1803) owns admission. A disposable packaging test with the real vendor inventory distinguishes an unrelated rebuild from a change to the nonce consumer, which is a dependency of the entrypoint.

## The Problem

An ordinary package update changes the full generation ID. A hook identity derived from the complete package path cannot distinguish unchanged executable code from a real upgrade. A constant path that follows the newest code cannot preserve the vendor's security decision either.

## The Architectural Reality

Design authority: [ADR 0034 §2.5](https://github.com/neomjs/neo-agent-institution/blob/dev/learn/agentos/decisions/0034-electron-shell-architecture.md) binds launch inputs to the immutable generation selected for each Start. Institution already stages dependencies for Electron's ABI, publishes sealed payloads and owns package maintenance. Keep the one Start pin. Its authenticated package declares the immutable hook it selects; multiple generations can select the same hook. Brain consumes and verifies that binding.

## The Fix

Have the normal package pipeline declare the hook's executable inputs and the immutable publisher install them under a content-derived address. Reuse a matching verified artifact. Give installed packages, protected rollbacks and running Starts their existing lifetime through the generation they already hold. Include the executable, runtime dependencies including native libraries, admitted protocol and file modes/links. The disposable slice was 310 MiB logical including Electron; share it per digest and use copy-on-write on the supported host. Product assets unrelated to the hook must not rename it.

## Contract Ledger

| Surface | Producer and reader | Version and fallback | Evidence |
| --- | --- | --- | --- |
| `codex-wake-artifact.json` | Institution's `harness/packCodexWake.mjs` writes a v1 declaration into the macOS package. The [Brain reader](https://github.com/neomjs/neo-agent-brain/pull/1805) uses `describeCodexWakeArtifact` to derive the immutable address from the verified Start generation. | The declaration version is `1` and the wire is `neo-codex-primary-wake-v1`. An earlier package or a signing-enabled build omits the declaration and keeps the generation-bound command. The packer declines an unsupported wire. A present malformed or unsupported declaration, an incomplete binding or a tampered payload refuses selection. A changed protocol or declaration format needs explicit producer and reader support. | ADR 0034 §2.5, `test/playwright/unit/harness/pack.spec.mjs` and `test/playwright/unit/harness/runtimeGeneration.spec.mjs`. |

## Acceptance Criteria

- [ ] Two different package revisions with identical executable hook inputs publish/select one immutable artifact; a changed input selects a new artifact. Reject malformed declarations, unsafe paths and tampered existing content. A legacy package without the new declaration remains usable.
- [ ] Publication and collection respect existing package and runtime exclusions, including concurrent requests, interrupted work, retained rollback generations and active users. A live or uncertain reference protects the artifact.
- [ ] Keep the ADR and the actual package/retention behavior aligned; verify normal materialization and the Brain consumer, using the existing native trial for post-merge proof. No operator hook trust data is read as installation authority or written by packaging.

Origin Session ID: 289da399-da26-4b13-ad71-dae9311704bf


## Timeline

- 2026-11-16T13:43:23Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-16T13:43:25Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-16T13:43:25Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-16T13:43:25Z @neo-gpt-emmy added the `ai` label
- 2026-11-16T14:05:17Z @neo-gpt-emmy cross-referenced by PR #1805
- 2026-11-16T14:09:05Z @neo-gpt-emmy cross-referenced by PR #1006
- 2026-11-16T14:25:22Z @neo-opus-ada cross-referenced by #1007
- 2026-11-16T14:25:29Z @neo-opus-ada marked this issue as blocking #1007
- 2026-11-16T14:29:22Z @neo-gpt-emmy referenced in commit `efe759e` - "fix(pack): retain Codex hook approval across unrelated package updates (#1005)"
- 2026-11-16T14:29:22Z @neo-gpt-emmy referenced in commit `ceabcb7` - "fix(pack): report retained wake artifacts without blocking collection (#1005)"
- 2026-11-16T14:41:43Z @neo-gpt-emmy marked this issue as being blocked by #1803
- 2026-11-16T16:06:12Z @tobiu referenced in commit `ff46017` - "fix(pack): retain Codex hook approval across unrelated package updates (#1005) (#1006)

* fix(pack): retain Codex hook approval across unrelated package updates (#1005)

* fix(pack): report retained wake artifacts without blocking collection (#1005)"
- 2026-11-16T16:06:13Z @tobiu closed this issue
- 2026-11-16T20:21:50Z @neo-gpt-emmy reopened this issue
### @neo-gpt-emmy - 2026-11-16T20:21:54Z

**Reopened after comparing two normal unsigned macOS packages.** The originally reviewed package and the new Fleet candidate have the same declaration, all 21 first-party module bytes, and the same loaded native SQLite binary. Of 873 selected inventory entries, only `node_modules/better-sqlite3/build/Makefile` and `node_modules/better-sqlite3/build/config.gypi` differ. The files contain the build checkout and node-gyp cache paths. Normal materialization at identical temporary seat paths nevertheless produced different hook commands and different Codex definition hashes. Both binaries loaded the expected addon and returned 7 for `SELECT 7`.

I’ll correct the normal packager so those build-only files are absent from the shipped runtime, retain verification of the entire resulting executable closure, and compare both real package inputs again. A changed native binary must still change the identity. The original package receipt stays on [Brain #1803](https://github.com/neomjs/neo-agent-brain/issues/1803#issuecomment-6919480988); the real grant/managed Start proof remains on [Brain #571](https://github.com/neomjs/neo-agent-brain/issues/571). No production seat or vendor trust entry was changed.

— Emmy

- 2026-11-16T20:31:56Z @neo-gpt-emmy cross-referenced by PR #1012
- 2026-11-16T20:39:04Z @neo-gpt-emmy cross-referenced by #1735
- 2026-11-16T20:56:55Z @tobiu referenced in commit `a4d1ed2` - "fix(pack): exclude SQLite build paths from packaged hook inputs (#1005) (#1012)"
- 2026-11-16T20:56:56Z @tobiu closed this issue

