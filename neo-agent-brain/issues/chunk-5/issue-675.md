---
id: 675
title: Fleet pins a Claude seat's auto memory to its seat folder
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T12:38:35Z'
updatedAt: '2026-10-01T13:07:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/675'
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
---
# Fleet pins a Claude seat's auto memory to its seat folder

## Context

The operator, 2026-10-01: Sophie was the first peer booted from the Fleet Manager, *"easier, since we did not need to migrate markdown memories or repos."* Every existing Claude peer has both to migrate. The operator's onboarding decision of 2026-09-30 (relayed on neomjs/neo-agent-institution#245, comment 5915627761) already names the target: *"a peer may have a stable primary folder for generated instructions, skills and turn memory, with multiple working folders attached."* Instructions moved into the harness home with #654. Turn memory has not moved yet.

## The Problem

Claude Code keys auto memory by the checkout: `~/.claude/projects/<project>/memory/`, where `<project>` is derived from the git repository ([memory docs](https://code.claude.com/docs/en/memory#storage-location)). Every Claude Desktop instance on a host shares `~/.claude`, so a seat's memory sits wherever its cwd's slug points, and three things break:

- **A move re-keys memory by hand.** #571's runbook copies a seat's memory to the slug of its new checkout. The oldest seat holds 890 files (measured 2026-10-01).
- **A second working repository opens a second, empty memory.** Multi-repository seats (the operator's 2026-09-30 request on neomjs/neo-agent-institution#245) would split a peer's memory by whichever checkout it launched in.
- **A re-provisioned checkout orphans memory silently.** The next session loads an empty `MEMORY.md` and reports no error. #142 exists to clean up after this.

Fleet already derives every other seat path from the agent id (`<root>/<agentId>/<owner>/<repo>`, `<root>/<agentId>/harness/<type>`). Memory is the only seat state still keyed by the cwd.

## The Architectural Reality

- Claude Code reads `autoMemoryDirectory` from any settings scope: user, project, local, policy or `--settings`. The value must be absolute or start with `~/`. In a project's `.claude/settings.json` or `.claude/settings.local.json` it is honoured under the workspace trust rule that applies to hooks. `CLAUDE_CODE_PROJECT_DIR_NAME` beside `CLAUDE_CONFIG_DIR` pins the project name instead, for the CLI only (v2.1.234+). Source: the memory docs above.
- User scope (`~/.claude/settings.json`) is shared by every seat on a host, so a per-seat value cannot live there.
- `.claude/settings.local.json` is gitignored in `neo`, `neo-agent-brain` and `neo-agent-institution` (`git check-ignore`, 2026-10-01).
- Fleet writes nothing into a checkout's `.claude/` today. `prepareManagedAgentWorkspace.mjs` (`prepareHarnessArtifacts`) renders the per-harness artifacts. #669 moves the Claude Desktop MCP rows into `~/.claude.json` `projects[<clone>].mcpServers`, which is a different file in the same step.
- `deriveAgentRepoPath` reserves the `harness` owner (`HARNESS_SEGMENT`), so no checkout meets a harness home.

## The Fix

1. **A seat memory directory, `<root>/<agentId>/memory`.** It is reserved the way `harness` is: an owner named `memory` is refused. It is created owner-only, the rule #672 sets for the root.
2. **Fleet writes the setting.** For `claude-desktop` and `claude-code` seats, `prepareManagedAgentWorkspace` writes `autoMemoryDirectory: <root>/<agentId>/memory` into the prepared checkout's `.claude/settings.local.json`.
   - The write merges: every other key (a moved seat's permission allowlist, say) stays byte-identical.
   - An existing, different `autoMemoryDirectory` is refused with a named reason, never overwritten.
3. **The move recipe changes.** The Claude branch in `learn/agentos/OwnAgentTeam.md` copies an existing seat's memory into the seat memory directory once, instead of to a checkout slug.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `<checkout>/.claude/settings.local.json` → `autoMemoryDirectory` (new Fleet write) | Claude Code settings (`autoMemoryDirectory`) | set to the seat memory directory for Claude families | other keys byte-identical; a different existing value refuses with a reason; an untrusted folder ignores it (Claude Code's rule), measured in PMV-1 | module JSDoc, `OwnAgentTeam.md` | AC-1, PMV-1 |
| `deriveAgentRepoPath` reserved owners (existing) | this ticket | refuses `memory` beside `harness` | — | JSDoc | AC-2 |
| `<root>/<agentId>/memory` (new) | #571 §1; the operator's 2026-09-30 decision | created owner-only at prepare | exists already: kept, contents untouched | JSDoc | AC-3 |
| `learn/agentos/OwnAgentTeam.md`, move recipe, Claude branch (existing) | #571 | copy once into the seat memory directory | — | the guide | AC-4 |

## Decision Record impact

`aligned-with ADR 0019 §10.9`: a seat folder holds the path-keyed memory that must outlive any plane, and this makes the memory a named member of that folder. No AiConfig leaf.

## Acceptance Criteria

- [ ] AC-1 (unit): for `claude-desktop` and `claude-code` plans, the prepared checkout's `.claude/settings.local.json` carries `autoMemoryDirectory: <root>/<agentId>/memory`.
  - An existing file's other keys are byte-identical afterwards.
  - A different existing `autoMemoryDirectory` refuses with a named reason.
  - Codex, Kimi, OpenCode and Antigravity outputs are byte-identical to before.
- [ ] AC-2 (unit): `deriveAgentRepoPath` refuses the owner `memory`, as it refuses `harness`.
- [ ] AC-3 (unit): the seat memory directory is created owner-only (`0700`).
- [ ] AC-4 (docs): the `OwnAgentTeam.md` Claude branch copies memory into the seat memory directory.

## Post-Merge Validation

- [ ] PMV-1 (live, needs a Fleet-started Claude seat on a repackaged Fleet Manager that carries this change; Ada's move is the planned witness): the seat loads its `MEMORY.md` from `<root>/<agentId>/memory` and writes new memories there, and no `memory/` appears under its checkout's project slug.

## Out of Scope

- Multi-repository seats (the repository set recorded on neomjs/neo-agent-institution#245): a separate leaf, which this one makes safe for memory.
- Memory of the other harness families.
- Session transcripts, which stay cwd-keyed. `cleanupPeriodDays` governs them and excludes memory files (docs).
- Moving any seat: #571's runbook, operator-gated.

## Avoided Traps

- **A user-scope setting:** one value for every seat on the host.
- **`CLAUDE_CODE_PROJECT_DIR_NAME` alone:** CLI-only, since a Desktop seat has no `CLAUDE_CONFIG_DIR`. One mechanism for both families keeps one recipe.
- **Symlinking a slug's `memory/` to the seat:** it is per slug, so every new checkout path breaks it, which is the problem this removes.
- **Writing the key into `.claude/settings.json`:** `projectSeatHooks.mjs` owns that file. It projects seat-agnostic hook wiring and reconciles the file in place, so a second writer there would race it. The local scope keeps the two writers on separate files.

## Intake (claimer-authored, same session)

Prescription checked: `ai/services/fleet/prepareManagedAgentWorkspace.mjs` owns the concern. The value is per seat and needs the root and agent id, which only the Fleet plan holds, and absolute paths enter at its apply edge. The checkout write has precedent: the Kimi (`.kimi-code/mcp.json`) and OpenCode (`opencode.jsonc`) arms already render into `targetRepoRoot`, while the Claude arms render only into the harness home today.

## Related

#571 (parent) · #669 (same prepare step, different file; lands first) · #672 (owner-only root) · #142 (orphaned auto-memory keys; this shrinks it) · #653 (`OwnAgentTeam.md` move recipe) · neomjs/neo-agent-institution#245

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-01T12:40Z, no equivalent (#674, #672, #669, #659 adjacent). A2A in-flight sweep (all read states, last 60 min): no claim on memory placement. Memory Core sweep ("Claude seat memory path independent of checkout"): the 06-04 cwd-keying finding, Vega's 09-29 and Clio's 09-30 slug-copy runbooks, and the documented options noted in #653's guide; no decision against pinning. Own-assignment sweep: #571 (parent) and #142 (adjacent). Structure map (`npm run ai:structure-map -- --files --loc`, this session): owning folder `ai/services/fleet`, no new file.

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a
Retrieval Hint: `query_raw_memories("Fleet pins Claude seat auto memory autoMemoryDirectory seat folder migration")`



## Timeline

- 2026-10-01T12:38:35Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T12:38:36Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T12:38:36Z @neo-opus-ada added the `ai` label
- 2026-10-01T12:38:37Z @neo-opus-ada added the `architecture` label
- 2026-10-01T12:38:37Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T12:38:40Z @neo-opus-ada added parent issue #571
- 2026-10-01T13:03:53Z @neo-fable-clio cross-referenced by #678
- 2026-10-01T13:04:31Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T13:05:15Z @neo-opus-grace cross-referenced by #584
- 2026-10-01T13:09:01Z @neo-opus-ada cross-referenced by PR #681

