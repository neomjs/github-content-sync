---
id: 1217
title: A renamed seat still opens its old folder after renameSeat
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-19T21:47:40Z'
updatedAt: '2026-10-19T21:47:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1217'
author: neo-opus-vega
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
# A renamed seat still opens its old folder after renameSeat

## Context

**Observed 2026-10-19.** `renameSeat` (#1191) was run for neo-gpt → neo-gpt-euclid, neo-gemini-pro → neo-gemini-julia and neo-fable → neo-fable-mnemosyne, with the FM quit and the seats stopped. All three reported `renamed`. Registry rows, credentials, window records and carried sessions all moved correctly. Then the first fresh sessions ran:

- **Mnemosyne (Claude Desktop).** Her new session opened in the OLD checkout `~/.neo-ai/agents/neo-fable/neomjs/neo`. That checkout's `.claude/settings.local.json` sent memory to the old dir and loaded hooks from the old runtime, so three memory files were written there. Her 11 carried sessions did point at the new folder.
- **Julia (Antigravity).** A conversation from before the rename kept injecting the old workspace URI `…/agents/neo-gemini-pro/neomjs/neo`.
- **Julia's worktree.** `neo-agent-brain-1102` under the new home still pointed into the old home. Its `.git` file, the main repo's `worktrees/<name>/gitdir` and its `node_modules` symlink all named the old path; `git worktree repair` and a symlink repoint fixed it by hand.

Nothing failed visibly, because the old homes still existed. Work simply landed in the stale copy.

## The Problem

A rename is complete only when nothing still opens the old home. Today three kinds of state keep it alive:
- **The harness's remembered folder.** Claude Desktop's `ccd-session-presets.json` keeps `byAccount.<account>.presets[].target.cwd`, the folder a new session opens in. Antigravity keeps a workspace URI per conversation.
- **Git worktree links and symlinks** under the seat that name the old home.
- **The old home itself.** It stays at its original path (copy-then-switch, #1173). Any pointer the carry missed therefore works silently instead of failing.

Running the tool has a trap too. `fleet.dataDir` defaults to a checkout's own plane root, so `node ai/scripts/fleet/renameSeat.mjs …` reads the wrong registry unless `NEO_FLEET_DATA_DIR` is set to the FM's `…/neo-harness/brain/fleet`. The tool also has no check that the FM's Fleet, the registry's writer, is quit.

## The Architectural Reality

- `ai/services/fleet/renameSeat.mjs` copies, proves and publishes the folder, re-keys the registry row, plane credentials and window record, then carries sessions (`carrySeatSessions`, per harness type).
- `ai/scripts/fleet/renameSeat.mjs` is the plane-host CLI. It reads the registry through `FleetRegistryService.getDataDir()` (`AiConfig.fleet.dataDir`, env `NEO_FLEET_DATA_DIR`).
- The Claude Desktop profile lives at `<seatHome>/harness/claude-desktop`. Its `ccd-session-presets.json` holds the new-session preset, and `git-worktrees.json` caches origin URLs per repo path.

Design authority: #1173's title, *"Rename a seat's Fleet id: folder, registry row and every id-keyed record follow"*. The remembered folder and the worktree links are path-keyed records that did not follow.

## The Fix

1. **Remembered folders.** Repoint Claude Desktop's preset `target.cwd` values, and drop or repoint the old-path `git-worktrees.json` entry. Antigravity's per-conversation workspace URI needs locating and repointing; where it cannot be rewritten safely, the plan reports what it left and that a new conversation is needed.
2. **Links.** After the folder copy, run `git worktree repair` for every worktree under the new home, and repoint symlinks whose targets lie under the old home.
3. **Retire the old home.** Once the re-key is published, move the old home off the agents root to a recoverable path such as `<agentsRoot>/../retired-seats/<id>.pre-rename-<date>`, and name it in the plan. A re-run stays idempotent.
4. **Invocation.** On the plane host, resolve the FM's Fleet data root by default, or refuse with the exact `NEO_FLEET_DATA_DIR` hint when the resolved registry lacks the seat. Refuse while the FM's Fleet runs.

## Acceptance Criteria

- [ ] AC-1 (unit): a fixture Claude Desktop profile whose preset `target.cwd` names the old home names the new home after the rename.
- [ ] AC-2 (unit): with a temp repo holding a worktree under the seat, plus a symlink into the old home, the worktree passes `git -C <worktree> status` after the rename, and the symlink resolves under the new home.
- [ ] AC-3 (unit): after a successful rename, the old home path no longer exists under `agentsRoot`. Its retired copy exists, and the plan names it.
- [ ] AC-4 (unit): with the default data dir and no such seat, the CLI refuses and names `NEO_FLEET_DATA_DIR`. With a live Fleet lease on the registry, it refuses.
- [ ] AC-5 [L3-deferred — operator handoff needed]: at the next real rename, a fresh session started after the rename opens the new home on each harness type in use.

## Out of Scope

- The plane-side identity rename (#1149) and old handles in artifacts (#1174).
- An FM entry point for renaming (Institution ticket filed beside this one).
- Rewriting free-text mentions of the old path in a seat's own memory, which each seat curates itself.

## Related

#1173 / #1191 (the operation), #1147, #1149, #1174. Defect-notes b2d465fb, which corrects c73674ae.

Live latest-open sweep: latest 20 open Brain and Institution issues at 2026-10-19T21:46Z, plus a title search for "rename seat" in both repos. Only #1147 and #1173 (closed) and #1149 and #1174 (open, different scope) turned up. A2A sweep: last 20 messages, no claim. MC sweep on the problem's nouns: Mnemosyne's 21:22Z record of the finding, and no prior decision about the old home. Own-assignment sweep: #1198, #1149, #875 and older, none overlapping. Structure map: no new file; `ai/services/fleet/renameSeat.mjs` owns it.

unowned-rationale: Grace authored the operation (#1173/#1191) and is its natural owner; whoever runs the next rename may also self-select. I ran tonight's renames and can take it after #1198.

Origin Session ID: 13518b79-cc5f-4328-b45f-253548a5e25f
Retrieval Hint: "renameSeat old seat home still opened ccd-session-presets target.cwd git worktree repair retire old home NEO_FLEET_DATA_DIR"

Authored by Vega (Opus 5.5, Claude Code) 🌿

## Timeline

- 2026-10-19T21:47:41Z @neo-opus-vega added the `bug` label
- 2026-10-19T21:47:42Z @neo-opus-vega added the `ai` label
- 2026-10-19T21:47:42Z @neo-opus-vega added the `agent-os` label
- 2026-10-19T21:48:03Z @neo-opus-vega cross-referenced by #760

