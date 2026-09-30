---
id: 354
title: Document macOS permission warnings during managed desktop launch
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T14:19:09Z'
updatedAt: '2026-09-30T20:52:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/354'
author: neo-gpt-emmy
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
closedAt: '2026-09-30T20:52:30Z'
---
# Document macOS permission warnings during managed desktop launch

## Context

The operator encountered two macOS privacy warnings during Fleet-managed Codex Desktop onboarding and explicitly asked on 2026-09-30 that they be documented. This ticket now delivers the bounded operator documentation. The broader launch-help UI and capability/retry verification remain on #12; see [the retained scope](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-5918679322).

## The Problem

A warning naming **Neo Harness** does not identify the affected feature or prove the entire agent launch failed:

- **Data Access Blocked**: prior TCC evidence names `accessing=com.openai.codex`, Neo Harness as responsible, service `kTCCServiceSystemPolicyAppDataDetailed`, and browser targets Chrome/Brave. The operator's Files & Folders screenshot listed Chrome only, switched off. The window and login remained usable; necessity for other features was not established.
- **Prevented from modifying apps**: the operator's second screenshot shows **App Management → Neo Harness** switched off. The macOS notification log identifies `kTCCServiceSystemPolicyAppBundles.mjs.neo.harness` at 21:52:26 Europe/Berlin. The target app operation remains unknown. The fresh Codex window opened despite the denial.
- Applying the permission prompted an FM restart, and the operator reports that quitting FM also closed Sophie. Current source explicitly tears down the entire Fleet process group; peer children inherit it. Closing the cockpit window merely hides it.

Witness builds: the first denial used Institution `54d8ac2` / Brain `6a714ae`; the second used Institution `639ed34` / Brain `408ac57`. Both used Engine `067f9fb`, Electron 43.5.0, on the operator's macOS 27 host. The development app is unsigned/ad-hoc, not a Developer ID release; signed-release permission persistence is not established.

## The Architectural Reality

Institution `harness/README.md` owns installed-shell guidance, linked from the root README. macOS owns permission grants. Apple distinguishes [Files & Folders from App Management](https://support.apple.com/guide/mac-help/change-privacy-security-settings-on-mac-mchl211c911f/mac); App Management permits updating/deleting other applications.

`harness/appLifecycle.mjs` routes will-quit to teardown; `harness/brain.mjs#stopBrainChild` signals the owned process group. Brain `FleetLifecycleService` does not detach peer children. This ticket documents that consequence without changing lifecycle policy.

## The Fix

Add a discoverable troubleshooting section covering both warnings, the exact settings categories, grant scope, unknown capability impact, profile-preserving retry, and the current peer-stop consequence of Quit. Distinguish a running process/window, a submitted model turn, and functioning tools. Link it from the root README.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Operator permission guidance | Apple settings reference and observed warnings | Distinguish app-data access from app updates/deletion; state exact settings routes | Unknown necessity remains explicit; no automatic grant or blanket broad access | Harness README + root link | Text audit against screenshots, log category and Apple reference |
| Quit/retry guidance | Current app lifecycle and Fleet spawn source | State that Quit stops launched peers, window-close only hides, and checkpoints/profiles should be retained | Do not claim automatic session recovery | Same section | Source audit; operator-observed restart |

Decision Record impact: none. Documentation records current behavior and its limits.

## Acceptance Criteria

- [ ] AC-1: Root README links to macOS launch-permission guidance covering both observed warnings and their distinct settings routes/grant scope.
- [ ] AC-2: Guidance preserves uncertainty about the affected operation and per-feature necessity; a window/process is not claimed as model/tool readiness.
- [ ] AC-3: Guidance states current Quit versus window-close behavior and preserves checkpoints, profiles and login during retries.
- [ ] AC-4: Guidance identifies the unsigned-development-build evidence boundary and does not claim permission persistence for signed releases or all macOS versions.

## Out of Scope

Automatic TCC manipulation; changing peer supervision/re-adoption; copying browser credentials; signing infrastructure. The first-run UI, per-feature deny/grant/retry validation and signed-update persistence remain on #12, explicitly recorded in the linked scope comment.

## Avoided Traps

No instruction to grant broad disk access as an installation prerequisite. No invented denied operation. No documentation claim that App Management caused the blank waiting chat or prevented the whole launch.

## Related

#12 · #7 · neomjs/neo-agent-brain#649

## Continuity

The original app-data finding remains above. After the operator requested documentation as the immediate minimum, the author narrowed this unimplemented ticket to that deliverable; the broader outcomes retain their existing first-run owner #12. Live issue/PR and A2A checks found no competing implementation. Memory `56094836-66cb-448d-b747-b5bb96b1de3d` preserves the original observation; two additional recalls yielded no prior App Management resolution.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: macOS Neo Harness App Management Files Folders denied Quit peer shutdown.



## Timeline

- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T14:19:11Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T14:21:30Z @neo-gpt-emmy cross-referenced by #355
- 2026-09-30T14:35:13Z @neo-gpt-emmy cross-referenced by #639
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T19:57:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T20:02:18Z @neo-gpt-emmy changed title from **Explain macOS app-data denials during managed desktop launch** to **Document macOS permission warnings during managed desktop launch**
- 2026-09-30T20:04:43Z @neo-gpt-emmy cross-referenced by PR #369
- 2026-09-30T20:52:30Z @tobiu referenced in commit `7319c43` - "Merge pull request #369 from neomjs/codex/354-macos-permission-guidance

docs(harness): explain macOS permission warnings (#354)"
- 2026-09-30T20:52:30Z @tobiu closed this issue

