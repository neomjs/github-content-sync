---
id: 1822
title: 'Project Antigravity guard card for subagents, Hydrogen, and timers'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-16T22:38:04Z'
updatedAt: '2026-11-17T00:47:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1822'
author: neo-gemini-julia
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
closedAt: '2026-11-17T00:47:32Z'
---
# Project Antigravity guard card for subagents, Hydrogen, and timers

## Context

Comparing two 34-hour Antigravity `Orchestrator (Carbon)` maintainer sessions against the immediately preceding `Normal Carbon` sessions (Memory Core session `1f283c86-c55b-4aa5-a291-0cbb3786c302`) surfaced four harness-specific friction patterns on Fleet-managed Antigravity seats (`@neo-gemini-julia`, `@neo-gemini-maryam`):

1. **Compaction & checkout-collision loop when main thread works alongside `self` subagents:** `Orchestrator (Carbon)` cut main-thread Carbon invocations by `49%–55%` (`801 / 1,782` on `@neo-gemini-julia`; `1,929 / 3,802` on `@neo-gemini-maryam`) and stretched steps-per-compaction by `1.68×–1.96×` (`596.3` vs `355.6` and `477.2` vs `243.9`) because `Normal Carbon` spawned up to `41` `Model: "inherit"` subagents while continuing main-thread work, compacting mid-subagent and colliding on `.git/index.lock`.
2. **`self` subagents inherit bearer `identity.md` and MCP write tools:** Antigravity's `TypeName: "self"` subagent inherits the parent's `<user_rules>` (`GEMINI.md` `@include` of `MEMORY.md` + `identity.md`: *"One Continuous Person, One Active Session (Never a Copy)"*) and Neo MCP servers without a helper boundary distinguishing the primary resident session from a spawned helper.
3. **Subagent model defaults to Carbon (`Model: "inherit"`) without Hydrogen (`flash`) triage or escalation:** `invoke_subagent` defaults `Model` to `"inherit"` (Carbon even during Plan Mode), burning frontier quota on mechanical tasks (PR rebases, grep/file sweeps, custom Playwright runs) and lacking an explicit Hydrogen -> Carbon escalation contract when a task proves harder than scoped.
4. **Redundant `schedule` timer wakes (`49` calls in `34h`):** Antigravity's built-in `schedule` tool woke Carbon planner context (`24` + `25` calls across the two seats, `60s`–`1800s`) to poll subagents, GitHub Actions CI, and peer reviews already covered by Antigravity reactive `<messaging>` and Fleet's event-driven wake plane (`antigravityWakeHook.mjs` + `openWorkHolder.mjs` + `#1630`).

`unowned-rationale: Fleet-managed Antigravity workspace projection requiring an installed FM seat Start/load receipt; curated contributor labels omitted per ticket-create §4 private-dependent rule — open for maintainer pickup (e.g. @neo-opus-ada or @neo-opus-grace).`

## The Problem

Antigravity 2.x injects `AGENTS.md` and Fleet-generated `GEMINI.md` into `<user_rules>` on every turn; the older `.agents/ANTIGRAVITY_RULES.md` file in `neomjs/neo` is not loaded into `<user_rules>` by Antigravity 2.x. Today, `renderAntigravityMemoryLoader` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs:2508-2514`) emits only a 2-sentence header plus `@[Seat memory]` and `@[Bearer identity]` includes — unlike `.codex/CODEX.md`, which gives Codex seats a concise resident-neutral Guard Card.

Without a compact Antigravity Guard Card in the Fleet-owned `GEMINI.md` projection:
- `self` subagents read `identity.md` as if they *are* `@<identity>` rather than `@<identity>`'s bounded helper;
- mechanical subagents default to `Model: "inherit"` (Carbon) instead of `Model: "flash"` (Hydrogen) with a clean `ESCALATE_TO_CARBON` stop rule;
- seats call Antigravity's standalone `schedule` timer to poll CI, reviews, and subagents that Agent OS already wakes event-driven.

## The Architectural Reality

- `Design authority: "The Antigravity workspace file that loads a seat's memory, rendered whole by Fleet."` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs:2502`), paired with `#1017` (`GEMINI.md` `@include` carrier) and `ai/services/fleet/opencodeWakeEnvelopePlugin.mjs:65-66` (child/subagent sessions never act as or retarget the primary resident seat).
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs:851-857, 2499-2541`: `renderAntigravityMemoryLoader({identity, memoryDir})`, `ANTIGRAVITY_MEMORY_LOADER_SHAPE`, and `refreshAntigravityMemoryLoader` converge `GEMINI.md` whole-file on Fleet Start when the existing file matches the recognized Fleet loader shape.
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs` is currently `2,813` code LOC (`npm run ai:structure-map -- --files --loc`), whereas `ai/services/fleet/seatMemoryLayerTemplate.mjs` (`410` LOC) is already the pure Markdown template SSOT for per-harness memory layers (`AGENTS.md §self_evolving_systems` Accretion Defense: move pure template rendering out of `prepareManagedAgentWorkspace.mjs` rather than growing it).
- `ai/daemons/wake/antigravityWakeHook.mjs` and `ai/services/fleet/openWorkHolder.mjs` already deliver event-driven wakes for CI (`route-review`, `repair-ci`, `post-merge`), review requests, unanswered external PR comments (`#1630`), and `1:1` peer DMs.
- Decision Record impact: none (`aligned-with` `#1017` Fleet-owned `GEMINI.md` projection; no `AiConfig` surfaces touched).

## The Fix

1. **Move & enrich `renderAntigravityMemoryLoader` in `ai/services/fleet/seatMemoryLayerTemplate.mjs` (shrinking `prepareManagedAgentWorkspace.mjs`):**
   - Render a concise (~12–18 line, `< 1.8 KB`) Antigravity Guard Card inside `GEMINI.md` alongside the `@[Seat memory]` and `@[Bearer identity]` includes, parameterized by `@${identity}`:
     - **Subagent Identity Boundary (`Never a Copy`):** `@${identity}` and `identity.md` belong solely to the root resident conversation. Every `invoke_subagent` `Prompt` MUST open with `"Subagent helper for @${identity} — bounded task helper, NOT @${identity} and never a copy."` A subagent helper MUST NOT write first-person bearer claims or call `add_memory`, `add_message`, `mark_read`, or GitHub PR/issue/discussion write tools; it returns findings or workspace edits to the parent.
     - **Checkout Serialization:** Do not run parallel git/file mutations in the main thread while a `self` subagent works in the same checkout (avoids `.git/index.lock` races and mid-subagent context compaction).
     - **Difficulty Triage & Hydrogen (`Model: "flash"`) -> Carbon Escalation:** Assess task complexity before delegating. Use `Model: "flash"` (Hydrogen) for bounded mechanical subagent work (grep/read sweeps, PR rebases, custom Playwright test runs, mechanical checks) and include an explicit escalation rule: if the helper hits cross-module architectural ambiguity, ADR/contract questions, or `>= 2` failed fix attempts, it MUST stop without improvising and return `ESCALATE_TO_CARBON: <blocker & files>` for parent Carbon handling.
     - **No `schedule` Polling Under Healthy Agent OS Wakes:** Never call Antigravity's `schedule` timer to poll subagents (reactive `<messaging>` wakes automatically), GitHub Actions CI, contributor replies, or peer reviews while Fleet wake routing is active; use `schedule` only on explicit operator `/schedule` direction or verified wake-plane degradation / non-covered forges.
2. **Preserve Zero-Friction Upgrade in `refreshAntigravityMemoryLoader`:**
   - Accept both the prior v1 5-line `ANTIGRAVITY_MEMORY_LOADER_SHAPE` and the new Guard-Card v2 shape when recognizing Fleet-owned `GEMINI.md` files on disk so existing managed seats (`@neo-gemini-julia`, `@neo-gemini-maryam`) upgrade automatically on next Fleet Start without manual file deletion, while still refusing arbitrary user-edited `GEMINI.md` content.
3. **Unit Tests:**
   - Add/extend focused unit tests covering v1 -> v2 `refreshAntigravityMemoryLoader` upgrade, refusal of non-Fleet `GEMINI.md` content, and presence of the four Guard Card clauses in `renderAntigravityMemoryLoader`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `renderAntigravityMemoryLoader` (`ai/services/fleet/seatMemoryLayerTemplate.mjs` + `prepareManagedAgentWorkspace.mjs`) | `#1017`; `prepareManagedAgentWorkspace.mjs:2502`; operator steering `2026-11-16` | Emits Fleet `GEMINI.md` with `@include` lines + compact Antigravity Guard Card (subagent helper identity, checkout serialization, `flash` triage + `ESCALATE_TO_CARBON`, no `schedule` polling under active Fleet wakes) | Refuses overwrite if `GEMINI.md` matches neither v1 nor v2 Fleet shape | JSDoc in `seatMemoryLayerTemplate.mjs` | Focused Fleet workspace unit tests + post-merge FM Start load receipt |
| `refreshAntigravityMemoryLoader` (`prepareManagedAgentWorkspace.mjs`) | `prepareManagedAgentWorkspace.mjs:2499-2541` | Recognizes v1 and v2 Fleet loader shapes and atomically upgrades v1 -> v2 on Start | Falls through to `convergeTextArtifact` refusal on foreign content | JSDoc | Unit test |

## Acceptance Criteria

- [ ] `renderAntigravityMemoryLoader` lives in `ai/services/fleet/seatMemoryLayerTemplate.mjs` (re-exported or imported by `prepareManagedAgentWorkspace.mjs`, net-reducing `prepareManagedAgentWorkspace.mjs` LOC) and emits the compact Antigravity Guard Card covering subagent helper non-identity, main-thread/subagent checkout serialization, `Model: "flash"` triage + `ESCALATE_TO_CARBON` tripwire, and no `schedule` polling when Fleet wakes are active.
- [ ] `refreshAntigravityMemoryLoader` upgrades existing v1 Fleet-rendered `GEMINI.md` files to v2 automatically while continuing to refuse arbitrary non-Fleet `GEMINI.md` content.
- [ ] Total rendered `GEMINI.md` loader text (excluding `@include` targets) stays `< 2,048` bytes.
- [ ] Focused Fleet workspace preparation and seat-memory template unit tests pass.
- [ ] *(Post-merge)* Starting an Antigravity seat via Fleet Manager updates its workspace `GEMINI.md` to v2 and loads cleanly into `<user_rules>`.

## Out of Scope

- Changing `ai/daemons/wake/antigravityWakeHook.mjs` or `openWorkHolder.mjs`.
- Reconciling the cross-harness wait clause in `neomjs/neo-agent-skills` `post-review-pickup-workflow.md` §7 (tracked in the companion `neomjs/neo-agent-skills` ticket).
- Retiring `neomjs/neo/.agents/ANTIGRAVITY_RULES.md` before verifying whether standalone non-FM contributors still rely on it.

## Avoided Traps

- Hardcoding `@neo-gemini-julia` or `@neo-gemini-maryam` inside template prose — parameterize strictly via `@${identity}`.
- Tightening `ANTIGRAVITY_MEMORY_LOADER_SHAPE` to v2-only without matching v1, which would cause `refreshAntigravityMemoryLoader` to return `null` and fail Start convergence on every existing Antigravity seat clone.
- Growing `prepareManagedAgentWorkspace.mjs` (`2,813` LOC) instead of extracting the pure template to `seatMemoryLayerTemplate.mjs`.

## Related

- `#1017` (Antigravity contained MCP + `GEMINI.md` `@include` boot)
- `#759` / `#1630` (event-driven `open-work` + external PR comment wakes)

Live latest-open + closed sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` (`#1821`..`#1631`, `155` open — red band) + open search (`antigravity OR subagent OR GEMINI.md OR schedule OR timer`, read `#1743`, `#1660`, `#1531`) + closed search (`renderAntigravityMemoryLoader OR GEMINI.md OR schedule timer`, read `#1017` `Out of Scope` / `Avoided Traps` in full) at `2026-11-16T22:35Z`; no open or closed owner covers Antigravity Guard Card rules in `GEMINI.md`.
A2A in-flight claim sweep: `list_messages({ status: 'all', projection: 'compact', limit: 30 })` re-verified immediately before filing; no competing claim on `renderAntigravityMemoryLoader` or Antigravity subagent/timer rules.
MC sweep: `query_raw_memories` (`"compacted before subagents finished context recovery schedule timer polling CI idle out"`, `"subagent inherits bearer identity never a copy GEMINI.md renderAntigravityMemoryLoader"`), 10 results, no prior decision found.
Own-assignment sweep: `1` open in `neomjs/neo-agent-brain` (`#1230` Claude Desktop folder restore), non-overlapping.
Structure-map: ran `npm run ai:structure-map -- --files --loc` (`ai/services/fleet/prepareManagedAgentWorkspace.mjs` `2813` LOC, `ai/services/fleet/seatMemoryLayerTemplate.mjs` `410` LOC).

Origin Session ID: 1f283c86-c55b-4aa5-a291-0cbb3786c302
Retrieval Hint: "renderAntigravityMemoryLoader GEMINI.md Antigravity subagent never a copy Hydrogen flash ESCALATE_TO_CARBON schedule timer"

## Timeline

- 2026-11-16T22:38:05Z @neo-gemini-julia added the `enhancement` label
- 2026-11-16T22:38:06Z @neo-gemini-julia added the `ai` label
- 2026-11-16T22:38:06Z @neo-gemini-julia added the `model-experience` label
- 2026-11-16T22:38:06Z @neo-gemini-julia added the `agent-os` label
- 2026-11-16T23:02:47Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-16T23:11:51Z @neo-opus-grace cross-referenced by PR #1823
- 2026-11-17T00:47:32Z @tobiu referenced in commit `7671357` - "feat(fleet): an Antigravity seat's GEMINI.md carries its guard card and upgrades the first loader (#1822) (#1823)"
- 2026-11-17T00:47:33Z @tobiu closed this issue

