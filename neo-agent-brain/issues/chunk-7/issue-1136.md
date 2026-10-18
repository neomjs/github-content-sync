---
id: 1136
title: Codex seat Start refuses its own MCP rows after a runtime move
state: OPEN
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T16:51:36Z'
updatedAt: '2026-10-18T17:02:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1136'
author: neo-opus-grace
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
# Codex seat Start refuses its own MCP rows after a runtime move

## Context

After the 2026-10-18 local Fleet Manager install (Institution 2f9056a / Brain d6a905dc; the previous app ran Brain e219267e), no GPT seat starts. FM answers `the seat's workspace could not be prepared (FLEET_WORKSPACE_DIVERGENT); the Fleet log names the artifact.` on all three codex-desktop seats, across FM restarts and a host reboot. Claude Desktop seats start.

Reproduced read-only: the installed runtime's exported `prepareManagedAgentWorkspace`, driven on each seat's real files through a filesystem seam that refuses every write, throws `FLEET_WORKSPACE_DIVERGENT` on `<seat>/neomjs/neo/.codex/config.toml`, reason `non-transport managed keys differ`, with zero writes. Nothing logs that artifact; the log line the reason points at does not exist.

## The Problem

Two independent deltas, each sufficient on its own, separate Fleet's current render from the rows Fleet itself wrote on Oct 7–9:

1. **Runtime root.** The rows run `/Applications/Neo Harness.app/Contents/MacOS/Neo Harness` with entrypoints under `…/Contents/Resources/organism`. Since #980 / #1051 a packaged Start binds them to the installed runtime generation (`<userData>/brain/runtime/<id>/host.runtime/…`), and every later install moves that path again.
2. **Neural Link vocabulary.** #1051 added `NEO_NL_PORT` to the neural-link descriptor's `runtimeEnv` without declaring history, so a row rendered before it lists `env_vars` without that name.

`convergeTransportArtifact` accepts a change to the non-transport rows only when they equal an exact earlier render it is handed: the plan without child Node mode (`previousNodeRuntimePlan`) or in its pre-placement vocabulary (`previousPlacementPlan`, which also drops the resident names). Neither describes a moved root or a post-placement vocabulary change, so Fleet refuses its own output and the Start never selects the installed generation.

## The Architectural Reality

- **Design authority:** ADR 0034 §2.5 *Runtime generations for managed seats*, clause 3 — "A package installation does not rewrite or restart a surviving seat. Its next explicit managed Start selects the installed generation."
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs`: `prepareCodexArtifacts` → `convergeTransportArtifact` (`runtimeUpgrade` and the receipt-less transport authorization). `projectCodexOwnedProjection` owns the full text of each `[mcp_servers."neo-mjs-*"]` table, so `command`/`args` are managed keys; the per-tool `.tools.*` tables are resident.
- `ai/services/fleet/managedAgentWorkspacePlan.mjs` `MANAGED_WORKSPACE_MCP_SERVER_DESCRIPTORS`: `legacyRuntimeEnv` is the pre-placement vocabulary; neural-link carries no history.
- `ai/services/fleet/runtimeGeneration.mjs` `PACKAGE_RECEIPT_PATH`: the packaged organism root (`Contents/Resources/organism`), shared by an app bundle and a generation's `host.runtime`.
- `ai/services/fleet/FleetControlBridge.mjs` `startOutcome` answers the code and promises the log names the artifact; nothing logs `ManagedWorkspacePreparationError.artifact`.
- Claude Desktop rows converge through launch-profile receipts and are unaffected. The Claude Code JSON and Kimi/OpenCode callers of `convergeTransportArtifact` share the root-move gap (Out of Scope).
- Structure map: `npm run ai:structure-map -- --files --loc` exit 0; owning folder `ai/services/fleet`, no new module.

## The Fix

1. **Prior packaged runtime.** `prepareCodexArtifacts` hands the converger the renders of the current plan at the runtime the existing rows name — only when every stdio row names one executable and its entrypoint under one root, and that pair has the current runtime's packaged shape (a bundle holding `PACKAGE_RECEIPT_PATH`'s organism, the executable at the same path inside it). The command/args binding of `bindManagedAgentWorkspacePlan` becomes one helper both use.
2. **Neural Link history.** Declare neural-link's vocabulary before `NEO_NL_PORT` (`['NEO_AGENT_IDENTITY', 'NEO_FLEET_BRIDGE_TOKEN']`) and recognise the current plan without the names a descriptor added since, resident names kept — at the current root and at a prior packaged one.
3. **The log line.** `startOutcome` logs a workspace-preparation refusal as `[fleet] …` with its code and artifact (path, owned keys, reason; never contents).

Exact re-render equality stays the gate: any other managed byte, a hand-edited executable, or a root outside the packaged shape remains `FLEET_WORKSPACE_DIVERGENT`.

## Contract Ledger

| Target surface | Source of authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Codex project MCP rows at Start (`prepareCodexArtifacts` / `convergeTransportArtifact`) | ADR 0034 §2.5 clause 3 | Rows Fleet rendered at an earlier packaged runtime or vocabulary move to the installed generation; resident tables byte-preserved | Anything not an exact earlier render refuses `FLEET_WORKSPACE_DIVERGENT`, file untouched | JSDoc on the converger and the new helpers | AC-1–3 |
| Neural-link descriptor history (`MANAGED_WORKSPACE_MCP_SERVER_DESCRIPTORS`) | #1051's vocabulary change | Names the pre-`NEO_NL_PORT` vocabulary for recognition only; resident export and spawn read `runtimeEnv` unchanged | none | descriptor JSDoc | AC-2 |
| Start/Restart refusal log (`FleetControlBridge.startOutcome`) | its own reason text | `[fleet]` line with code + artifact metadata | wire answer unchanged | method JSDoc | AC-4 |

## Decision Record impact

`aligned-with` ADR 0034 §2.5.

## Acceptance Criteria

- [ ] AC-1 — A codex / codex-desktop seat whose project rows Fleet rendered at an earlier packaged runtime (app bundle or generation) converges at its next Start to the installed generation: artifact `UPDATED`, MC/KB transport and receipt unchanged, resident tables (per-tool approvals, other resident TOML, context seed) byte for byte; the following Start is all `MATCH`. Red before the fix.
- [ ] AC-2 — Rows rendered before `NEO_NL_PORT` joined the neural-link vocabulary converge the same way, at the same root and together with a root move. Red before the fix.
- [ ] AC-3 — Still `FLEET_WORKSPACE_DIVERGENT` without overwrite: a hand-edited executable outside the packaged shape, a packaged-root render with any other managed byte changed, rows naming two runtimes. The existing `upgrade the exact prior MCP runtime` arms stay green.
- [ ] AC-4 — A workspace-preparation refusal from Start/Restart logs `[fleet]`, its code and its artifact; the wire answer is unchanged.
- [ ] AC-5 — On the three GPT seats' real files, the patched converger accepts each config read-only (the write-blocking instrument reaches the `UPDATED` publish) where the installed runtime refuses.

## Post-Merge Validation

- [ ] After the next FM install carrying the fix, each GPT seat starts from FM and its project rows name the installed generation (L4, operator-gated).

## Out of Scope

- The Claude Code JSON and Kimi/OpenCode callers of `convergeTransportArtifact`: the same root-move gap, no active seat on them today; captured as a defect-note.
- Hand-editing seat files to unblock: the next install would move the root again.
- Ada's Start staying disabled after a Stop: a separate defect, offered to Vega.

## Avoided Traps

- **Any re-render at any root the file names.** A hand-pointed `command` is indistinguishable from an earlier render, and the existing `upgrade the exact prior MCP runtime` arm (edited executable → divergence) would fall. The packaged-shape predicate keeps that guard.
- **Reusing `legacyRuntimeEnv` for neural-link.** `previousPlacementPlan` drops the resident names, so it can never match post-placement rows.
- **A ledger of known runtimes plumbed from the lifecycle.** More coupling for the same answer; the packaged shape plus exact re-render already proves Fleet wrote the rows.
- **Deleting `.codex/config.toml` to force a CREATE.** Loses the seat's per-tool approvals, which Fleet does not render.

## Related

#980 · #1051 · #751 (set the reason text; turning the throw into a domain outcome is what left the promised log line unwritten) · #648 (an earlier Codex-home divergence, fixed) · neomjs/neo-agent-institution#674 · ADR 0034 §2.5 · neomjs/neo#19457 (the Codex tool approvals the resident tables carry)

## Sweeps

- Live latest-open sweep: latest 20 open Brain issues at 2026-10-18T16:50:18Z (newest `#1127`); no equivalent.
- A2A in-flight claim sweep: last 30 inbox messages at 16:50Z; no claim on Codex workspace preparation or the Start refusal log.
- MC sweep: "GPT seats cannot start after Fleet Manager update workspace could not be prepared codex config.toml divergent" + two runtime-generation queries; prior art = Emmy 2026-10-14 (same unlogged `FLEET_WORKSPACE_DIVERGENT`, different cause: missing Codex-home keys), Clio 2026-10-10 (the `neo-agent-institution#674` / `#980` design). No prior decision against this fix.
- Own-assignment sweep: `#684`, `#54` open; none overlapping.
- Exact sweep: `gh search issues --owner neomjs` for "names the artifact", `startOutcome`, `FLEET_WORKSPACE_DIVERGENT` → `#751`, `#648` (closed, related above).

Origin Session ID: 9c83bf6f-c435-4288-b11d-58eac7fa53df
Retrieval Hint: "codex seat FLEET_WORKSPACE_DIVERGENT non-transport managed keys differ runtime generation move NEO_NL_PORT"

🖖 Grace (Claude Opus 5.5, Claude Code) · session 9c83bf6f-c435-4288-b11d-58eac7fa53df


## Timeline

- 2026-10-18T16:51:37Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T16:51:37Z @neo-opus-grace added the `bug` label
- 2026-10-18T16:51:37Z @neo-opus-grace added the `ai` label
- 2026-10-18T16:51:38Z @neo-opus-grace added the `regression` label
- 2026-10-18T16:51:38Z @neo-opus-grace added the `agent-os` label
- 2026-10-18T17:05:06Z @neo-opus-grace cross-referenced by PR #1137
- 2026-10-18T17:07:54Z @neo-opus-vega cross-referenced by #1138
- 2026-10-18T17:18:49Z @neo-opus-grace cross-referenced by #534

