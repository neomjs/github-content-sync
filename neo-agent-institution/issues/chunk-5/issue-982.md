---
id: 982
title: A retained wake helper blocks a Fleet Manager update
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T04:01:12Z'
updatedAt: '2026-11-15T04:23:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/982'
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
closedAt: '2026-11-15T04:23:10Z'
---
# A retained wake helper blocks a Fleet Manager update

## Context

The supported macOS installer refused the rehearsed Fleet package at Institution `195de6e1` during a real `--dry-run --quit --open`. It classified two Antigravity wake sidecars as off-canonical Fleet apps. Both executables were Electron Helpers beneath a retained runtime generation, with Antigravity language-server parents. The dry run made no changes.

**Design authority.** #674 requires the installer to count users of the app bundle while allowing processes pinned to retained generations to survive. #473 protects running canonical, renamed and staging app copies. The live outcome belongs to #732 and #12.

## Problem and fix

`harness/install.mjs#runningHarnessPaths` matches any path component named `Neo Harness*.app`. A macOS payload published as `host.runtime` still contains `Contents/Frameworks/Neo Harness Helper.app/Contents/MacOS/Neo Harness Helper`. The inner name trips the same refusal as a second Fleet app. The existing generation test checks the main executable, whose path contains no inner helper app.

Teach the installer’s process census to distinguish retained runtime helpers from users of a replaceable bundle. Scope recognition to the selected instance’s genuine generation and its packaged helper. Keep unknown or ambiguous candidates in the refusal set. Use the same decision during planning, while waiting for quit and before touching a package slot.

| Surface | Authority | Behavior | Evidence |
|---|---|---|---|
| Installer process census | Kernel executable paths, selected runtime root, existing generation layout and verifier | Exclude a proven retained helper. Preserve canonical, renamed and staged app users. | Focused installer and runtime-generation specs, actual read-only dry run |
| Package replacement | Existing transaction, custody, slot and continuity guards | Keep the independent protections in place; no process is killed to pass the census. | Pre-mutation recheck and ordinary install/restore controls |

## Acceptance criteria

- [ ] A retained helper with the actual nested macOS name does not block the installer for the selected runtime root. Duplicate PIDs running the same executable do not change the decision.
- [ ] Helpers inside the canonical bundle and genuine renamed or staging bundles remain visible. Foreign, linked/escaping or malformed generation lookalikes do not gain an exemption.
- [ ] Planning, quit waiting and the last process check before a slot mutation agree. A newly observed app user still blocks.
- [ ] The supported dry run advances past this specific false refusal with the real enrolled sidecars still alive. Record any later independent refusal separately. Actual installation remains on the operator’s cut.

**Out of scope.** Stopping seats, changing vendor settings, loosening package-slot locks or retirement rules, and altering the separate Claude rotation command.

Decision Record impact: aligned with the existing runtime-generation update contract (ADR 0034 §2.5).
Live latest-open + closed sweep: latest 20 open, 56 open (red); searched running-off-canonical, retained generation and installer. #473/#674 supply the two invariants. #732 owns the wider outcome. No overlapping claim in the recent 30 all-state mailbox entries, own open assignments or targeted memory sweep. Already owned and blocking a real cut.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: retained host.runtime Antigravity sidecar nested Electron Helper package census

## Timeline

- 2026-11-15T04:01:12Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T04:01:14Z @neo-gpt-emmy added the `bug` label
- 2026-11-15T04:01:14Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T04:01:14Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T04:01:14Z @neo-gpt-emmy added the `build` label
- 2026-11-15T04:11:07Z @neo-gpt-emmy cross-referenced by PR #983
- 2026-11-15T04:23:10Z @tobiu referenced in commit `929e04f` - "fix(installer): keep retained wake helpers outside the package census (#982) (#983)"
- 2026-11-15T04:23:11Z @tobiu closed this issue

