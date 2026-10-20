---
id: 1223
title: A Claude Desktop seat starts in the folder its app remembers
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T09:38:36Z'
updatedAt: '2026-10-20T12:07:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1223'
author: neo-opus-vega
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-20T12:07:51Z'
---
# A Claude Desktop seat starts in the folder its app remembers

## Context

Operator, 2026-10-20: "FM sets the starting folder for seats. this is a BUG when pointing to wrong folders."

Live case: Mnemosyne's seat (`neo-fable-mnemosyne`, Claude Desktop) has not started a session since the 10-19 rename. Her app's `neo` preset still names `~/.neo-ai/agents/neo-fable/neomjs/neo`, which was moved to the Trash, so every new session fails. The Desktop log shows "Working directory no longer exists" at 09:09:03, 09:17:16 and 09:18:33. Her seat lease (`harness/claude-desktop/.neo-fleet-seat-lease.json`) records the right `cwd` (`…/neo-fable-mnemosyne/neomjs/neo`); nothing hands it to the app.

It is not new. The first Fleet-launched Claude Desktop seats each needed a manual folder step: Ada on 09-30 ("sign-in, the folder open"), and Clio on 10-09, who booted with no folder in a scratch workspace and so skipped her project settings, hooks and skills (#571 comment 6073068185). Each time it was treated as an operator step.

## The Problem

The Fleet decides each seat's checkout but imposes it only on Codex Desktop. A Claude Desktop seat starts in whatever folder its app profile remembers: a preset seeded once and never converged. Any drift survives every Start: a rename, a moved or deleted checkout, a hand edit, or a fresh profile with no folder at all. #1217 (PR #1220) repoints presets at rename time. Only a convergence at Start makes the FM the authority for where a seat works.

## The Architectural Reality

- `ai/services/fleet/deriveHarnessLaunchSpec.mjs`: `codex-desktop` launches with `--user-data-dir=<profile> --open-project=<cwd>`, and a missing or relative `cwd` throws ("silently opening the Fleet server's cwd would fork project-keyed memory"). `claude-desktop` launches with `--user-data-dir=<home>` plus `CLAUDE_USER_DATA_DIR`, and no folder.
- `ai/services/fleet/carrySeatSessions.mjs` (`carryClaudeDesktopRecords`) is the only writer of `<instanceHome>/ccd-session-presets.json`, and it runs only during `renameSeat`. `git grep ccd-session-presets` on `dev` finds no Start path.
- The lease already records the provisioned `cwd`.
- A running app owns its state files: a write to `ccd-session-presets.json` while the app runs is reverted on its next use (observed 10-19 23:12, reverted by 10-20 09:09). At Start, before spawn, the app is not running.
- Design authority: `deriveHarnessLaunchSpec.mjs`'s codex-desktop rule (a GUI seat opens its provisioned checkout, never an incidental folder), and #571's terminal predicate (seats launch into the layout the Fleet provisions).

## The Fix

Before spawning a `claude-desktop` seat, converge its remembered starting folder to the provisioned checkout. Every preset `target.cwd` in `<instanceHome>/ccd-session-presets.json` that doesn't name the checkout gets rewritten, using the rehome primitive `carrySeatSessions` already has, applied at Start while the app is stopped. If no preset exists, seed one at the checkout so a fresh profile never boots with no folder. If the installed Claude Desktop accepts a starting-folder argument the way Codex takes `--open-project`, prefer it and keep the file convergence as the fallback. None is documented in the launch spec today, so verify it against the installed build before relying on it.

## Acceptance Criteria

- [ ] AC-1 (unit): a `claude-desktop` Start whose preset names a stale folder rewrites it to the provisioned checkout before spawn. A preset that already names the checkout leaves the file byte-identical.
- [ ] AC-2 (unit): a profile with no preset file gets one preset at the checkout before spawn.
- [ ] AC-3 (unit): a missing or relative provisioned checkout refuses the Start before spawn, as `codex-desktop` does.
- [ ] AC-4 [L3]: Mnemosyne's seat, whose preset names the trashed folder, starts from the FM and opens a session in `~/.neo-ai/agents/neo-fable-mnemosyne/neomjs/neo` with no manual edit.

## Out of Scope

Rename-time carry (#1217, #1220), Antigravity's per-conversation workspace URI, and Codex Desktop (already explicit).

## Related

#571 (parent), #1217, #1220, #1191, Institution #732.

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-20T09:37Z, plus title searches ("starting folder", "claude desktop cwd preset") in Brain and Institution. Closest is #1217 (rename time only); #571 is the parent epic. A2A sweep: Julia's #1220 claims rename time only, and no Start-time claim exists. MC sweep: Ada's 09-30 and Clio's 10-09 first starts, both with a manual folder step, never ticketed.

handoff: offered to @neo-gemini-julia (team Brain & FM, holds #1217/#1220, the nearest code). Vega is on team 3D today.

Origin Session ID: 2eade974-a9ae-4fc9-9362-0b70b91b7410
Retrieval Hint: "Claude Desktop seat starting folder preset cwd Fleet Start converge open-project"

Authored by Vega (Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-10-20T09:38:37Z @neo-opus-vega added the `bug` label
- 2026-10-20T09:38:38Z @neo-opus-vega added the `ai` label
- 2026-10-20T09:38:43Z @neo-opus-vega added parent issue #571
- 2026-10-20T09:46:58Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T10:32:38Z @neo-gemini-julia cross-referenced by PR #1225
- 2026-10-20T11:10:28Z @neo-gemini-julia referenced in commit `a6d8ad4` - "fix(fleet): converge only local presets, skip when no account, and log Start outcome (#1223)"
- 2026-10-20T12:07:51Z @tobiu referenced in commit `0837560` - "fix(fleet): converge claude-desktop session presets to checkout before start (#1223) (#1225)

* fix(fleet): converge claude-desktop session presets to checkout before start (#1223)

* fix(fleet): converge only local presets, skip when no account, and log Start outcome (#1223)"
- 2026-10-20T12:07:51Z @tobiu closed this issue

