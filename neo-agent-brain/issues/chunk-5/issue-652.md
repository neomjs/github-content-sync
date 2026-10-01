---
id: 652
title: 'OwnAgentTeam.md carries the move recipe: an existing Claude Code or Codex agent joins a Fleet seat without losing its memories'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-09-30T20:58:55Z'
updatedAt: '2026-10-01T08:18:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/652'
author: neo-fable-clio
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
closedAt: '2026-10-01T08:18:12Z'
---
# OwnAgentTeam.md carries the move recipe: an existing Claude Code or Codex agent joins a Fleet seat without losing its memories

## Context

Operator direction, 2026-09-30: most outside operators will arrive with ONE Claude Code or Codex agent already running on the harness's default data dir, and the Fleet Manager must let that agent's markdown memories move into its Fleet seat — a repo guide first, an optional step in the agent-create flow later. The same evening the team planned the move of two of its own seats (neomjs/neo-agent-brain#571's checklist) and measured what a Fleet seat changes: a NEW checkout path and a NEW harness profile, while nothing the harness keys by path follows on its own.

`learn/agentos/OwnAgentTeam.md § Isolate Memory Correctly` already states the mechanism (Claude Code file-memory is keyed by the project cwd; `--user-data-dir` does not redirect it). It says how to *isolate* teammates; it does not say how to *move* one in. The recipe was verified on this machine tonight and lives nowhere an operator can read it.

## The Problem

An operator who registers their existing agent in the FM and clicks Start gets a fresh clone at `<NEO_FLEET_AGENTS_ROOT>/<id>/<owner>/<repo>` and a fresh profile at `<id>/harness/<type>`. The agent boots with an empty memory index, no permission allowlist, and a signed-out harness — and every file it had is still on disk under the old path, silently orphaned. Without the recipe the operator either loses the agent's memory or gives up on the FM.

## The Architectural Reality

Measured 2026-09-30 on the installed FM (Institution `639ed34`, Brain `408ac57`):

- Every Claude Desktop instance on a machine shares `~/.claude` (Claude Code's config dir; Fleet's launch env keeps `HOME`, `ai/services/fleet/FleetLifecycleService.mjs:77`). Claude Code keys three things by the cwd, slugged with every non-alphanumeric character → `-` (a space included): `~/.claude/projects/<slug>/memory/` (the markdown memories), `~/.claude/projects/<slug>/*.jsonl` (transcripts), and `~/.claude.json` → `projects["<cwd>"]` (allowed tools, MCP toggles, trust). The checkout-local `.claude/settings.local.json` carries the permission allowlist; a seat's `.env` retires under Fleet, which holds the PAT (operator ruling 2026-09-27, #571).
- A Fleet seat's cwd is `deriveAgentRepoPath` (`<root>/<id>/<owner>/<repo>`), its profile `deriveAgentInstanceHome` (`<root>/<id>/harness/<type>`, passed as `--user-data-dir` + `CLAUDE_USER_DATA_DIR`, `deriveHarnessLaunchSpec.mjs:322`); Fleet writes the MCP config into that profile (`prepareManagedAgentWorkspace.mjs`), so the operator signs in once and never copies an MCP config.
- Codex keys its trust entries by path in `~/.codex/config.toml` (#571 body); where Codex keeps per-project memory is the Codex seat's fact to state, not this ticket's — see the ledger.
- `git clone` refuses a non-empty directory: nothing may be pre-created under the seat folder; checkout-local files are copied after the first Start clones.
- Today an FM Quit stops the peers it launched (documented in Institution `harness/README.md § macOS permissions`, PR neomjs/neo-agent-institution#369); the recipe's first launch is a witness run with the old launch path kept intact.

## The Fix

One new subsection in `learn/agentos/OwnAgentTeam.md`, under *Isolate Memory Correctly*: **Bringing an existing agent into a Fleet seat**. Shape:

1. What follows automatically (nothing path-keyed) and what Fleet provides (the profile's MCP config, the identity, the PAT).
2. The per-harness table of path-keyed surfaces — Claude Code: memory dir, transcripts (optional), the `~/.claude.json` project entry, `.claude/settings.local.json`; Codex: the trust entry and its memory location as verified by the Codex seat.
3. The recipe, copy-never-move: register without starting → compute the new cwd and slug → `rsync -a` the memory dir + `diff -rq` → clone the `~/.claude.json` entry (backup first; the file is rewritten whole by every running instance) → Start, sign in, copy `settings.local.json` into the clone → one verification turn (the identity line, the memory path, a witness file that lands in the NEW dir and not the old) → rollback = the old launch, untouched → retire the old dir weeks later with a pointer file, never delete.
4. The two current caveats, linked not restated: an FM Quit stops launched peers (Institution README), and a fresh seat's wake route is separate work (neomjs/neo-agent-brain#79).

No new file, no generated file; the section joins the guide that already owns the mechanism.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `OwnAgentTeam.md § Bringing an existing agent into a Fleet seat` | this ticket; #571 §2 (one layout, path-keyed memory re-keyed, never orphaned) | states the path-keyed surfaces per harness and the copy-never-move recipe with its verification and rollback | — | the section itself | the recipe run on this machine 2026-09-30: 890 + 33 memory files copied and `diff -rq` identical at their new slugs |
| Claude Code slug rule as documented | observed `~/.claude/projects` slugs (7 on this machine, one with a space in the path) | `cwd.replace(/[^A-Za-z0-9]/g, '-')` | if a future Claude Code changes the rule, the guide's "look for the dir Claude created on first start" fallback still works | same section | the seven slugs listed in the PR body |
| Codex rows (trust entry, memory location) | the Codex seat (@neo-gpt-emmy) verifies before the PR opens | the guide names the exact paths or says "unverified" for that row | an unverified row is stated as such, never guessed | same section | the Codex seat's readback quoted in the PR body |

## Acceptance Criteria

- [ ] AC-1 The subsection exists under *Isolate Memory Correctly*, names every path-keyed Claude Code surface with its exact location and the slug rule, and says which of them the recipe copies and which it leaves.
- [ ] AC-2 The recipe is copy-never-move with a `diff -rq` proof, a stated order (before the first session on the new cwd; checkout-local files after the clone), a one-turn verification and a rollback; nothing in it deletes.
- [ ] AC-3 The Codex rows are either verified by the Codex seat (quoted in the PR) or explicitly marked unverified; no guessed Codex path.
- [ ] AC-4 The two caveats (Quit stops launched peers; fresh-seat wake route) are linked to their owning artifacts, not restated; the guide's TD Mermaid (if any) renders; `npm run lint` in a Skills checkout is not touched (no skill files change).

## Out of Scope

The optional import step in the FM's agent-create flow (a later Institution + Brain leaf once the guide has been used by one outside operator); export/import of agent setups across machines; the agents-root placement (#571's open sub); the wake route (neomjs/neo-agent-brain#79); the Quit survival contract (Institution #12).

## Avoided Traps

- Moving instead of copying (a failed first launch would orphan the memories twice). Pre-creating anything under the seat folder (the clone refuses). Editing `~/.claude.json` without a backup while instances run. Guessing Codex's memory layout from Claude's. Restating the Quit caveat instead of linking the README that owns it.

## Related

neomjs/neo-agent-brain#571 (parent; its checklist is the source) · neomjs/neo-agent-brain#79 · neomjs/neo-agent-institution#12 · neomjs/neo-agent-institution#369 · neomjs/neo-agent-institution#351 (the wizard epic records the operator's direction: guide first, form later)

Live latest-open sweep: the latest 20 open Brain issues read at 2026-09-30T20:57Z (#650 … #514); no equivalent. A2A claim sweep (last 30, all read-states, 19:36–20:53Z): no claim on this scope. Memory Core rationale sweep: only tonight's move runbook and Vega's 2026-09-29 / 2026-06-04 memory-layer notes — no prior guide decision. Own-assignment sweep: Institution #351, neo#19058 — neither is this. Epic layer: #571 is the parent (5 subs today). Structure map: N/A (docs, existing file).
Decision Record impact: none (aligned-with ADR 0019 §10.9 as #571 states it).

Origin Session ID: ca4b10cc-1608-4154-9732-eff2324831ea
Retrieval Hint: "existing agent into Fleet seat memories copy never move slug ~/.claude/projects settings.local.json OwnAgentTeam"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session ca4b10cc-1608-4154-9732-eff2324831ea

## Timeline

- 2026-09-30T20:58:55Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-30T20:58:56Z @neo-fable-clio added the `documentation` label
- 2026-09-30T20:58:57Z @neo-fable-clio added the `enhancement` label
- 2026-09-30T20:58:57Z @neo-fable-clio added the `ai` label
- 2026-09-30T20:58:57Z @neo-fable-clio added the `agent-os` label
- 2026-09-30T20:59:07Z @neo-fable-clio added parent issue #571
- 2026-09-30T21:03:47Z @neo-fable-clio cross-referenced by PR #653
- 2026-09-30T22:14:00Z @neo-fable-clio cross-referenced by #656
- 2026-09-30T22:31:28Z @neo-fable-clio referenced in commit `96b56d4` - "docs(agentos): the Codex rows of the move recipe carry a Codex maintainer's readback (#652)

Codex memory is per instance under $CODEX_HOME/memories/ and project trust is a config.toml table keyed by the absolute checkout path; the recipe names both and keeps the rest of the home out of scope."
- 2026-09-30T22:42:55Z @neo-fable-clio referenced in commit `4000615` - "docs(agentos): the move recipe branches by harness family and names each seat's real home (#652)

Four Fleet families, four homes: claude-desktop keeps the default config root, claude-code moves it with CLAUDE_CONFIG_DIR, codex-desktop reads codex-home inside its home, codex reads the home itself. The Claude derivation is scoped to the measured Desktop setup and the documented overrides are named; the Codex copy goes into the seat's home before the first Start, which inspects only the checkout path."
- 2026-09-30T23:16:47Z @neo-fable-clio referenced in commit `a9ee97b` - "docs(agentos): the project-entry step names each Claude family's own config file (#652)

The Desktop family edits ~/.claude.json; the CLI family's file lives under its relocated config root as <CLAUDE_CONFIG_DIR>/.claude.json, which Fleet's launch contract documents. The step, the surfaces table and the harness table say so; backup and copy semantics unchanged."
- 2026-09-30T23:37:43Z @neo-fable-clio referenced in commit `2fad6dd` - "docs(agentos): the project-entry clone reads its source file and writes the seat's file separately (#652)

A relocated CLI config root holds no old entry, so a same-file jq wrote null; the step now reads the old agent's file, writes the branch's own file with its other fields kept and its backup taken, and stops on a missing source entry. Executed against fixtures: two files, the same-file Desktop control, a missing source."
- 2026-10-01T08:18:12Z @tobiu closed this issue
- 2026-10-01T08:18:13Z @tobiu referenced in commit `b4af5dd` - "Merge pull request #653 from neomjs/clio/652-existing-agent-into-fleet-seat

docs(agentos): OwnAgentTeam.md carries the move recipe for an existing agent joining a Fleet seat (#652)"
- 2026-10-01T09:09:05Z @neo-fable-clio cross-referenced by #659
- 2026-10-01T10:34:10Z @neo-fable-clio cross-referenced by #571
- 2026-10-01T10:52:45Z @neo-fable-clio cross-referenced by #669

