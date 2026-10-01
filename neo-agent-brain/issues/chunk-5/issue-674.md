---
id: 674
title: A Codex seat's MCP switch is shadowed by the Fleet's project table
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T12:35:48Z'
updatedAt: '2026-10-01T13:14:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/674'
author: neo-opus-grace
commentsCount: 0
parentIssue: 659
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T13:14:50Z'
---
# A Codex seat's MCP switch is shadowed by the Fleet's project table

## Context

The operator asked for MCP servers that can be switched on and off inside the harness (#659). On a Fleet-launched Codex seat the switch exists and does nothing. Measured on the installed Codex desktop app (26.928.31416, `com.openai.codex`; #659 comment 5931565012): the switch sends `config/value/write` with `keyPath: mcp_servers.<name>.enabled` and `filePath: null`, which writes the user layer, `$CODEX_HOME/config.toml`, the seat's `codex-home/config.toml`.

## The Problem

The Fleet renders every `neo-mjs-*` table into the project layer, `<clone>/.codex/config.toml`, with `enabled = <matrix>` (`prepareManagedAgentWorkspace.mjs` `renderCodexMcpTable`). Project outranks User (`ConfigLayerSource::precedence`, 25 > 20), so the switch's value never wins. The installed CLI shows it (`codex mcp list --json`, codex-cli 0.159.2, throwaway `CODEX_HOME` and project):

| project table | home table | effective |
|---|---|---|
| `enabled = true` | `enabled = false` | **on**: today's Fleet seat after a switch |
| no `enabled` | `enabled = false` | off |
| no `enabled` | none | on |

A second defect sits on the same line: convergence owns each table body whole, `enabled` included, so a cockpit change of the matrix on an existing Codex seat fails its next Start with "non-transport managed keys differ".

## The Architectural Reality

- `prepareCodexArtifacts`: the project file converges through `convergeTransportArtifact` (owned projection = the `neo-mjs-*` table bodies, `projectCodexOwnedProjection`; a tenant seat also holds a transport receipt over the Memory Core and Knowledge Base tables). The home file converges through `convergeTextArtifact`, which owns three keys, plus `convergeCodexRemoteTrust`.
- Codex deep-merges the layers (`codex-rs/config/src/merge.rs`). **A home-only `[mcp_servers."…"] enabled = false` is not safe to seed:** when the project layer is not loaded (an untrusted project, or the seat's Codex opened on another folder), Codex reads it as a server without a transport and refuses to load any configuration ("failed to load bootstrap configuration … invalid transport", measured with the same CLI).
- Fleet Codex seats: Sophie today; more as seats move to Fleet (#571).

## The Fix

### Contract Ledger

| Target Surface | Source of Authority | Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `renderCodexMcpTable` — project `mcp_servers."neo-mjs-*".enabled` | The measured layer precedence above; this ticket's Fix 1 | Fleet-off writes `enabled = false`; Fleet-on omits the key so the seat's own switch decides. | A home switch cannot override Fleet-off. Without a home switch, an omitted key defaults on. | Renderer JSDoc | AC-1; AC-2 with the installed Codex parser |
| Codex-home `mcp_servers` switch values | The measured bootstrap failure above; this ticket's Fix 2 | Fleet never seeds a switch into the home; existing seat-written switches remain untouched. | Home-only server tables without a loaded project transport remain an upstream limitation. | Codex preparation JSDoc | AC-2 |
| `convergeCodexProjectSwitches` and the transport receipt | The existing convergence contract; this ticket's Fix 3 | Converge only project `enabled` lines before transport convergence, including previous Fleet `enabled = true` rows. Re-issue a receipt that authenticated the previous tables for the converged tables before publishing them. | Unrelated managed edits still refuse. Receipt re-issue requires a hash match to the pre-convergence tables. | Convergence JSDoc | AC-3; AC-4; interrupted-publication retry and divergence controls |

## Acceptance Criteria

- [ ] AC-1: a fresh Codex seat's project tables carry `enabled = false` for each server the Fleet switches off and no `enabled` for the others; the home holds no `mcp_servers` table.
- [ ] AC-2: a switch written to the home survives the next Fleet Start untouched, and the installed Codex parser reads it.
- [ ] AC-3: a cockpit change of the matrix applies at the next Start without a divergence refusal.
- [ ] AC-4: a seat rendered by the previous Fleet (`enabled = true` lines, tenant receipt) starts without a divergence refusal, with only `enabled = false` lines left in its project tables.

## Out of Scope

- The Claude half of #659 (the toggle store in `~/.claude.json`) and Fix 4 (`github-workflow` on by default): both follow #669. Until Fix 4, a seat turns GitHub on from the cockpit; the app's switch can turn on only what the Fleet leaves on.
- Codex's own behaviour when its switch writes a home-only table and the project layer is not loaded: upstream.
- Other harness renderers (Kimi, OpenCode, Claude): byte-identical.

## Decision Record impact

`none`.

## Related

Parent #659. #669 edits the same module's Claude Desktop renderer; this ticket touches only the Codex functions and the transport receipt's file-name constant. #649 (Codex home trust convergence).

## Sweeps

Live latest-open sweep: latest 20 open Brain issues at 2026-10-01T12:35:21Z, no equivalent; `gh search issues` "codex enabled project config shadow": none. A2A in-flight sweep (latest 12:29Z): no claim on this scope; Euclid's #669 boundary leaves the toggle slice with #659's owner. MC sweep: covered by #659's filing; no decision on the Codex layer. Own-assignment sweep: #659 is the parent, #672 and #517 touch other surfaces, #16 is the Codex cwd defect, which shares the module but not the concern. Structure map (this session, exit 0): owning folder `ai/services/fleet`; no new file.

Body revised 2026-10-01 ~13:00Z: the home seed of the first draft was measured unsafe (the bootstrap failure above) and replaced by Fix 1–3.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "Codex MCP switch shadowed by project layer enabled; enabled=false only for Fleet-off servers; home-only mcp_servers table breaks Codex bootstrap"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-01T12:35:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T12:35:50Z @neo-opus-grace added the `bug` label
- 2026-10-01T12:35:51Z @neo-opus-grace added the `ai` label
- 2026-10-01T12:35:51Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T12:35:55Z @neo-opus-grace added parent issue #659
- 2026-10-01T12:38:36Z @neo-opus-ada cross-referenced by #675
- 2026-10-01T12:52:37Z @neo-opus-grace cross-referenced by #659
- 2026-10-01T12:53:05Z @neo-opus-grace cross-referenced by PR #676
- 2026-10-01T13:03:53Z @neo-fable-clio cross-referenced by #678
- 2026-10-01T13:04:31Z @neo-fable-clio cross-referenced by #679
- 2026-10-01T13:05:15Z @neo-opus-grace cross-referenced by #584
- 2026-10-01T13:14:50Z @tobiu referenced in commit `5cd89d7` - "fix(fleet): a Codex seat's own MCP switch decides every server the Fleet leaves on (#674) (#676)

The Codex app's switch writes mcp_servers.<name>.enabled into the seat's Codex home (the user
layer), and the Fleet's project tables carried enabled = <matrix>, which outranks it. A project
table now says enabled = false only for a server the Fleet switches off; the others carry no
enabled key. The Fleet never seeds the home: a home-only table breaks Codex's bootstrap whenever
the project layer is not loaded.

The enabled lines converge before the table bodies, so a cockpit change of the matrix lands
instead of failing the next Start as divergence, and a seat rendered by the previous Fleet migrates
in one Start; a tenant seat's transport receipt is re-issued for the converged tables first."
- 2026-10-01T13:14:51Z @tobiu closed this issue
- 2026-10-01T13:33:31Z @neo-opus-grace cross-referenced by #687
- 2026-10-01T15:00:31Z @neo-gpt-emmy cross-referenced by PR #681
- 2026-10-01T15:15:27Z @neo-fable-clio cross-referenced by #696
- 2026-10-01T16:14:41Z @neo-fable-clio cross-referenced by PR #703
- 2026-10-01T17:40:56Z @neo-gpt cross-referenced by PR #698

