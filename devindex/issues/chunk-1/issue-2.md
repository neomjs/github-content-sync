---
id: 2
title: A committed launch config pins the dev server to one agent's clone
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-08-20T12:07:08Z'
updatedAt: '2026-08-24T03:37:03Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/2'
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
closedAt: '2026-08-20T15:07:35Z'
---
# A committed launch config pins the dev server to one agent's clone

## Context

`.claude/launch.json` is committed and names one machine-specific absolute path:

```json
"runtimeArgs": ["--prefix", "/Users/Shared/github/neomjs/devindex", "run", "server-start"]
```

`/Users/Shared/github/` is the operator's workspace. Every agent maintainer has a separate root — `/Users/Shared/claude/`, `/Users/Shared/opus-vega/`, `/Users/Shared/fable/`, `/Users/Shared/clio/`, `/Users/Shared/codex/`, `/Users/Shared/antigravity/` all exist on this machine and each holds its own checkout.

Surfaced while setting this repository up as a working clone after the operator's 2026-08-20 direction that tickets and PRs now get created here too. Until that change, one hardcoded path was harmless; it is not any more.

Live latest-open sweep: 1 open issue at time of filing (#1, column resize); no equivalent. A2A in-flight claim sweep over the latest 30 messages: no overlapping `[lane-claim]`.

## The Problem

The failure is not "the preview does not start". It is worse: **the preview starts, and serves the wrong tree.**

Any maintainer whose clone is not at `/Users/Shared/github/neomjs/devindex` launches a dev server rooted in the **operator's** checkout. Their own edits are invisible, the operator's uncommitted state is what renders, and nothing in the output says so. A change that was never applied reads as applied-and-ineffective; a bug already fixed locally reads as still broken.

This is the same class as the warning already carried in `neomjs/neo`'s e2e config, where `reuseExistingServer: false` is justified in a comment as: an already-listening server from a foreign clone satisfies the readiness URL and silently serves the wrong tree — *"false reds AND, worse, false greens."* The same hazard, arriving through a committed config rather than a squatting port.

There is a second, quieter effect: two agents previewing at once both drive the operator's tree on the same port, so they collide with each other and with him.

## The Architectural Reality

- `.claude/launch.json` — committed, so the value is shared by every clone; the `--prefix` argument is what pins the root
- `package.json` → `server-start` = `webpack serve -c ./node_modules/neo.mjs/buildScripts/webpack/webpack.server.config.mjs --open`, which is root-relative and needs no prefix when invoked from the repository itself
- The harness resolves `launch.json` relative to the open workspace, so the entry is already running in the right directory before `--prefix` overrides it

## The Fix

Drop the `--prefix` pair so the command runs in whatever clone the harness opened:

```json
"runtimeArgs": ["run", "server-start"]
```

If a prefix is genuinely wanted, it has to be derived at launch rather than committed — but the plain form needs no derivation, because the working directory is already correct.

## Acceptance Criteria

- [ ] `.claude/launch.json` contains no absolute filesystem path
- [ ] Starting the preview from a clone at any root serves **that** clone — verified by an edit that is visible in the served page and by the server's reported content root
- [ ] Two clones can run the preview without both resolving to the same directory

## Out of Scope

- Port-collision handling between simultaneous previews; `autoPort` already covers the port, and the directory is the defect here
- Any change to `server-start` itself

## Related

- neomjs/devindex#1 — filed while setting up the clone that surfaced this

Origin Session ID: 3e4f33e0-fb23-4a61-a2a0-7f396950f3d6


## Timeline

- 2026-08-20T12:07:10Z @neo-opus-grace added the `bug` label
- 2026-08-20T12:07:10Z @neo-opus-grace added the `ai` label
- 2026-08-20T14:38:44Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-20T14:42:45Z @neo-opus-grace cross-referenced by PR #3
- 2026-08-20T15:07:35Z @tobiu closed this issue
- 2026-08-20T15:07:35Z @tobiu referenced in commit `cada592` - "Merge pull request #3 from neomjs/fix/2-launch-config-clone-agnostic

fix(harness): the dev server runs in the clone that opened it (#2)"
### @neo-gpt - 2026-08-24T03:19:08Z

[acceptance-probe][neomjs/neo#17420]

Rebased github-workflow ToolService explicitly targeted `neomjs/devindex`. The assignee guard first refused a blind add on #2 with `ASSIGNEE_CONFLICT`; this same comment was then updated through repository validation → identity guard → target association → mutation. No deployment config was changed.

- 2026-08-24T03:42:21Z @neo-gpt cross-referenced by PR #17673
- 2026-08-29T13:04:42Z @neo-opus-grace cross-referenced by PR #12

