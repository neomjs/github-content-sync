---
id: 900
title: The cockpit does not say how far the installed app is behind dev
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-09T12:42:23Z'
updatedAt: '2026-11-09T12:42:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/900'
author: neo-fable-mnemosyne
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
---
# The cockpit does not say how far the installed app is behind dev

> Filed by 🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 7eade7d6-fb66-4acd-ac18-a13d83035071

## Context

Specimen, 2026-11-09. A bridge fix merged on Oct 31 (neomjs/neo-agent-brain#1390: resume a live Memory Core session instead of replacing it). Nine days later the operator's Claude seats still replaced their session after transport errors: 90 renewals in the machine's bridge log since Nov 1, no resume. The installed app's receipt names Brain `4ca0f506` (Oct 30), which predates the fix. That revision is 75 commits behind `dev`, 29 of them fixes; the installed Institution revision is 29 commits behind. Nobody knew, because finding out took a shell, the receipt file and three `git` commands.

`#846` was the same class one week earlier (the installed app lacked a merged window-placement fix).

## The Problem

"Merged" and "running on this machine" are different states, and the product shows only the first one's absence by its symptoms. A seat that hits a fixed bug re-diagnoses it. An operator has no way to see that an update would end a known failure.

## The Architectural Reality

- The installed app keeps a receipt (`.package-slots.json` in the harness runtime folder) with `owners.brain.revision`, `owners.engine.revision` and `owners.product.revision`.
- The plane's containers carry their own Brain revision, which the cockpit's deployment-state read already reaches. An app ahead of the plane, or behind it, is a second split nobody sees.
- `apps/agentos/view/system/` is the System view; no file under `apps/` reads the receipt today (searched `package-slots`, `owners.brain`, `installedRevision`, `commits behind`: none).

Design authority: `ROADMAP.md`, "Source merged, package loaded and behaviour validated stay distinct states"; row 5, "the vessel is updated … the product returns to `live` by its own guidance".

## The Fix

One row per owner in the System view: the revision, its commit date, and "n commits behind dev" when a comparison is available. The same for the plane's Brain revision, beside the app's. Information only: no banner, no gate, no update action.

The receipt and the plane revision are local facts. Who reads the comparison (the fleet server, the shell) is the implementer's call; without it the row says why in words.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 The System view shows the installed app's Institution, Brain and engine revisions with their commit dates, read from the app's own receipt.
- [ ] AC-2 With a comparison available each row reads "n commits behind dev". Without one it names the reason. It never shows zero for unknown.
- [ ] AC-3 The plane's Brain revision stands beside the app's, so a split between them is visible.
- [ ] AC-4 Unit: fixture receipt and comparison answers for current, behind and not compared.
- [ ] AC-5 (post-merge, installed) On a candidate older than `dev` the operator reads the lag without a terminal. Receipt on row 5's walkthrough.

## Out of Scope

- Updating without closing a harness (`#732`), and how a deployment receives merged code (`neo-agent-brain#77`).
- Which of the missing commits matter: no changelog in the cockpit.
- Restarting long-lived processes after an update (`neo-agent-brain#121`).

## Avoided Traps

- **A notification.** The lag is a fact to read when something misbehaves, not an interruption.
- **Counting from tags.** The fix that is missing is usually one that never reads as release-worthy (`neo-agent-brain#77`'s own finding).

## Related

#846 · #732 · #516 · neomjs/neo-agent-brain#121 · neomjs/neo-agent-brain#77 · neomjs/neo-agent-brain#1390

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open at 11:06Z and the newest 8 re-read at 12:39Z, plus searches here and in `neo-agent-brain` (`installed behind dev`; `update available revision`; `installed candidate stale pin`); nearest are `#846` (one fix), `#732` and the two Brain epics above; no equivalent found.
A2A sweep: latest 12 messages, all read-states, at 11:26Z; no claim on this scope.
MC sweep: "a fix was merged but the installed app kept running old code; nobody noticed the installed version was behind", 5 results (failed installs and roll-backs), no prior decision on a lag readout.
Own-assignment sweep: 5 open here, none on this surface.

Origin Session ID: 7eade7d6-fb66-4acd-ac18-a13d83035071
Retrieval Hint: "installed app behind dev", "merged fix not installed", "package-slots receipt revision"

## Timeline

- 2026-11-09T12:42:24Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-09T12:42:25Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-09T12:42:25Z @neo-fable-mnemosyne added the `help wanted` label
- 2026-11-09T12:42:25Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-09T12:42:26Z @neo-fable-mnemosyne added the `ai` label

