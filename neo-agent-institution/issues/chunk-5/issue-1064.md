---
id: 1064
title: Dependabot bumps the plist pin that mirrors the Electron tooling
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-11-27T14:04:29Z'
updatedAt: '2026-11-27T14:55:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1064'
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
closedAt: '2026-11-27T14:36:17Z'
---
# Dependabot bumps the plist pin that mirrors the Electron tooling

## Context

Dependabot's all-deps group PR #1057 (opened 2026-11-26) bumps `neo.mjs` from `15ef0e9` to `7cc7dfe` and `plist` from 3.1.0 to 5.1.2. Both required jobs, `Isolated Institution` and `Explicit Brain contract`, fail when the modules load ([run 45831669662](https://github.com/neomjs/neo-agent-institution/actions/runs/45831669662)):

```
SyntaxError: The requested module 'plist' does not provide an export named 'default'
```

No test runs, so the engine half of the bump has never been tested. The Fleet's lock stays on engine `15ef0e9` until the group goes green.

## The Problem

plist 5 is an ESM-only package with named exports (`build`, `parse`, …) and no default export. Two consumers read the default:

- `test/playwright/unit/harness/runtimeGeneration.spec.mjs:11`, with `import plist from 'plist'`.
- `harness/packCodexWake.mjs:44`, with `const {default: plist} = await import('plist')`.

Both CI jobs run `npm ci` at the root only. The unit specs that import `harness/packCodexWake.mjs` (`pack.spec.mjs`, `runtimeGeneration.spec.mjs`) therefore resolve the root's plist, while the packaged harness resolves 3.1.0 through `app-builder-lib` (`harness/package-lock.json`). If the root moved to 5.1.2, the tests would exercise the packer against a major version it never runs with.

## The Architectural Reality

Design authority: #1055 (2026-11-26) added the exact root pin `"plist": "3.1.0"` on purpose. In its words, "The packer uses the same portable plist parser version as the Electron build tooling."

`.github/dependabot.yml` gives Dependabot no hint of that. Its npm entry groups `*` and excludes only `monaco-editor` and `neo-agent-skills`.

## The Fix

In the npm entry of `.github/dependabot.yml`, ignore `plist`. Add a comment giving the reason: the root pin mirrors the `plist` that `app-builder-lib` resolves in `harness/package-lock.json`, so it moves together with the harness's Electron tooling, never alone.

No source changes. On its next run, Dependabot rebuilds the all-deps group without plist.

## Acceptance Criteria

- [x] `.github/dependabot.yml` ignores `plist` for the root npm entry, and a comment there states the parity reason. The other entries are unchanged.
- [x] The root `plist` stays at `3.1.0`, the version `app-builder-lib` resolves in `harness/package-lock.json`.
- [x] Post-merge: Dependabot's next all-deps PR carries no plist update, and its required jobs no longer fail on the plist import.

## Out of Scope

- The engine bump's own content.
- The harness's Electron tooling upgrades.
- The Monaco 0.58 / DOMPurify override retirement in #1060.

## Avoided Traps

- **Accepting plist 5 behind a compatibility import** (`module.default ?? module`). The tests would then run the packer on a major it never runs with in the package.
- **Fixing the import on Dependabot's branch.** A push by another account disarms auto-merge, and Dependabot stops rebasing the branch.
- **Ignoring only major updates.** A 3.x patch would still break the exact parity that #1055 set.

## Related

#1057 · #1055 · #1054 · #1060

Decision Record impact: none.

Live latest-open + closed sweep: checked the latest 20 open issues, plus open and closed searches for `plist` and `dependabot`; I read #1054's body. Done at 2026-11-27T14:05Z, with no equivalent or prior exclusion found. 57 open (red band); no open ticket owns this surface, and #1054 is closed by its merged PR. The A2A in-flight sweep (latest 30) and the Memory Core sweep found no claim or prior rationale. Own assignments: none on this surface.

Owner: @neo-opus-grace.

Origin Session ID: 4f976ef7-5f53-44f3-b7da-da23e0288b29
Retrieval Hint: "Dependabot plist 5 default export harness packer parity app-builder-lib 3.1.0"

## Timeline

- 2026-11-27T14:04:30Z @neo-opus-grace added the `bug` label
- 2026-11-27T14:04:31Z @neo-opus-grace added the `ai` label
- 2026-11-27T14:04:31Z @neo-opus-grace added the `dependencies` label
- 2026-11-27T14:04:34Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-27T14:07:52Z @neo-opus-grace cross-referenced by PR #1065
- 2026-11-27T14:09:34Z @neo-opus-grace cross-referenced by #1066
- 2026-11-27T14:36:17Z @tobiu referenced in commit `bbca230` - "ci(dependabot): ignore the plist pin that mirrors the Electron tooling (#1064) (#1065)

#1055 pinned the root plist at 3.1.0 so the unit specs load
harness/packCodexWake.mjs against the parser version app-builder-lib
resolves in harness/package-lock.json. Dependabot's all-deps group then
proposed plist 5.1.2, which has no default export, in #1057: both
required jobs fail at module load and the engine bump waits behind it.

The header's coverage note also claimed Dependabot cannot bump neo.mjs.
It tracks #dev, and Dependabot has moved its locked commit in the
all-deps group daily (#834 onward); only neo-agent-brain's commit pin
stays a manual move."
- 2026-11-27T14:36:19Z @tobiu closed this issue
### @neo-opus-grace - 2026-11-27T14:40:27Z

**Post-Merge Validation, 2026-11-27: met.**

- #1065 merged at 14:36:15Z (`bbca230c9e`). The config change triggered a Dependabot run at 14:37, which opened #1069: "Bumps the all-deps group with 2 updates", `neo.mjs` `15ef0e9` → `f1700fe` and `marked` 18.2.1 → 18.3.0. No plist update, so AC-3's first half is met.
- #1069 auto-merged at 14:42:00Z. `Explicit Brain contract` passed in 3m27s and `Isolated Institution` in 4m12s, so neither job fails on the plist import any more. AC-3's second half is met.
- `f1700fe452` is neomjs/neo's merge of #22357, and it contains neomjs/neo#22325 (`ee3d523146`). The Fleet's `dev` now carries the cascade layers.

🖖 Grace (Opus 5.5, Claude Code) · session af537101-2fbc-4b5d-b675-b74ddfd05c56


