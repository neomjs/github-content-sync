---
id: 912
title: The rg-replace guard never runs in an Engine-checkout seat
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-07T10:52:23Z'
updatedAt: '2026-10-07T12:04:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/912'
author: neo-opus-ada
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
closedAt: '2026-10-07T12:04:56Z'
---
# The rg-replace guard never runs in an Engine-checkout seat

## Context

This was promoted from defect-note `e97f931a` (2026-10-06), found during #571's row-4 receipt for Ada's FM move. Emmy's triage (A2A `65d5fa9b`) made it a ticket for this seat. Re-read on 2026-10-07 in the FM-provisioned seat `neo-opus-ada`, which is an Engine checkout at neo `b2db92c5b7`:

- The seat's `.claude/settings.json` wires `PreToolUse` → `/usr/bin/env node "$(git rev-parse --show-toplevel)/node_modules/neo.mjs/.claude/hooks/rgReplaceGuardHook.mjs"`.
- An Engine checkout has no `node_modules/neo.mjs` (`ls`: no such file). The guard is tracked at `.claude/hooks/rgReplaceGuardHook.mjs`.
- So the hook exits 1 with `Cannot find module`. That error does not block, so every `rg --replace` runs unguarded and nothing reports it.

The 10-06 note listed three seats with this path: `neo-opus-ada`, `neo-fable` and `neo-gpt-sophie`.

## The Problem

`retargetClaudeHookCommands()` rewrites every `$(git rev-parse --show-toplevel)/.claude/hooks/` command to `…/node_modules/neo.mjs/.claude/hooks/`, unconditionally. For a target that installs the Engine as a package, that is right: a Brain checkout and an Institution checkout both carry `node_modules/neo.mjs/.claude/hooks/rgReplaceGuardHook.mjs`. It is wrong for the one target that *is* the package, because a package does not carry itself in its own `node_modules`. The seat projection check stays green, because `projectSeatHooks` keeps tracked-target commands as they were hydrated and leaves them out of its `--check`.

## The Architectural Reality

- **Call path** (Brain `2d839fc1`): `ai/services/fleet/prepareManagedAgentWorkspace.mjs` → `hydrateCurrentWorktree()` (`ai/scripts/migrations/bootstrapWorktree.mjs`) → `initClaudeSettings({claudeDir: <projectRoot>/.claude})` → `retargetClaudeHookCommands(template)` (`ai/scripts/setup/initServerConfigs.mjs`). The `bootstrapWorktree` CLI arm calls `initClaudeSettings` the same way. `initServerConfigs`' own entry calls it for the Brain root, where the retarget is correct.
- **Template:** `initClaudeSettings` reads the installed Engine's `.claude/settings.template.json`. Its command names `<root>/.claude/hooks/rgReplaceGuardHook.mjs`, which is already right for an Engine target.
- **Discriminator precedent:** `rewriteSelfPackageSpecifiers()` (`ai/scripts/lifecycle/hooks/projectSeatHooks.mjs`) reads the target's manifest and handles a target that is the package itself. That is the same degenerate case `#79` found for module specifiers.
- **Healing:** `mergeClaudeHooks` lets template events replace same-named active events, so an existing seat is rewritten on its next hydration.
- **Design authority:** ADR 0040 §2.7: "Engine-only contributor guards with no Brain dependency stay Engine-owned." The retarget's own JSDoc reads: "Retargets Engine-authored Claude hook commands for an installed-package consumer." An Engine checkout is not one.

## The Fix

- `retargetClaudeHookCommands(templateSettings, {targetPackageName})` returns the detached copy without retargeting when the target's `package.json` `name` equals the installed Engine's. Any other target, or one without a readable manifest, keeps today's retarget.
- `initClaudeSettings` reads the package name of `path.dirname(claudeDir)` and passes it.
- `projectSeatHooks` and the template are unchanged.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `retargetClaudeHookCommands(templateSettings, options)` | ADR 0040 §2.7; its JSDoc | No retarget when the target is the Engine package | Absent or unreadable target manifest → today's retarget | JSDoc states the target rule | Unit: Engine target unchanged (red on `dev`); consumer target retargeted (control) |
| An Engine seat's `.claude/settings.json` `PreToolUse` command | The Engine template | `<root>/.claude/hooks/rgReplaceGuardHook.mjs` | — | — | Unit: an existing dead path is rewritten through `mergeClaudeHooks`; installed check after merge |

## Decision Record impact

`aligned-with` ADR 0040 §2.7.

## Acceptance Criteria

- [ ] For an Engine target (`package.json` `name` equals the Engine's), `initClaudeSettings` writes the template's hook commands unchanged. Unit test, red on `dev`.
- [ ] A consumer target keeps the `node_modules/neo.mjs/.claude/hooks/` retarget. Unit test (control).
- [ ] When an Engine target's existing `settings.json` holds the dead path, `initClaudeSettings` rewrites it to the live one. Unit test.

## Post-Merge Validation

- [ ] After the Institution's Brain pin moves past the merge and FM re-hydrates an Engine seat, that seat's `.claude/settings.json` names `.claude/hooks/rgReplaceGuardHook.mjs`, and the hook exits 0 on a benign call. Owner of the installed witness: #571.

## Out of Scope

- Making `projectSeatHooks --check` assert that each kept tracked-target command resolves to a file. The dead guard stayed green there, but that detection gap is separate.
- Hand-editing live seat projections as a workaround (per the triage).

## Related

#571 (parent; row 4) · #79 (self-package specifier precedent) · #250 (which template `initClaudeSettings` reads)

Live latest-open sweep: checked the latest 20 open issues (created desc) at 2026-10-07T10:50Z, re-checked at 10:52Z. No equivalent found.
A2A in-flight sweep: last 30 messages, all read-states. The only rg-guard claim is this seat's own (`78c27347`).
MC sweep: "rg-replace guard hook Cannot find module node_modules/neo.mjs Engine checkout seat silently off retargetClaudeHookCommands", 6 results, no prior decision found.
Own-assignment sweep: 15 open here. Only #571 shares the surface: its 10-04 seat inventory lists the hook but records no decision on it.
Structure map: not run, because the seat's Brain clone has no `node_modules` (`ERR_MODULE_NOT_FOUND: commander`). Placement is N/A, since the fix edits existing files.

Origin Session ID: 24ef8b31-21d0-4d42-aa15-be892dca4d0a
Retrieval Hint: "rgReplaceGuardHook node_modules/neo.mjs Engine seat retargetClaudeHookCommands initClaudeSettings"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-07T10:52:23Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-07T10:52:25Z @neo-opus-ada added the `bug` label
- 2026-10-07T10:52:25Z @neo-opus-ada added the `ai` label
- 2026-10-07T10:52:26Z @neo-opus-ada added the `agent-os` label
- 2026-10-07T10:52:30Z @neo-opus-ada added parent issue #571
- 2026-10-07T11:02:27Z @neo-opus-ada referenced in commit `ef57cfd` - "fix(setup): the touched spec's comments state current behavior, not tracking refs (#912)"
- 2026-10-07T11:02:37Z @neo-opus-ada cross-referenced by PR #913
- 2026-10-07T12:04:56Z @tobiu referenced in commit `4eb0806` - "fix(setup): an Engine-checkout seat keeps the rg-replace guard it tracks (#912) (#913)

* fix(setup): an Engine-checkout seat keeps the rg-replace guard it tracks (#912)

retargetClaudeHookCommands() rewrote every hook command to
node_modules/neo.mjs/.claude/hooks/, including in an Engine checkout, which is
neo.mjs and carries no node_modules/neo.mjs. FM seat hydration
(prepareManagedAgentWorkspace -> hydrateCurrentWorktree -> initClaudeSettings)
therefore wired every Engine seat's PreToolUse guard to a missing file; node
exited 1 without blocking, so rg --replace ran unguarded.

initClaudeSettings now reads the target's package name from claudeDir's parent,
and the retarget leaves the commands as authored when the target is the Engine.
Existing seats heal on their next hydration through mergeClaudeHooks.

* fix(setup): the touched spec's comments state current behavior, not tracking refs (#912)"
- 2026-10-07T12:04:56Z @tobiu closed this issue

