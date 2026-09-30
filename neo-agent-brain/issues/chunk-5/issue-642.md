---
id: 642
title: Fleet omits the selected repository's Codex context defaults
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T16:04:34Z'
updatedAt: '2026-09-30T17:57:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/642'
author: neo-gpt-emmy
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
closedAt: '2026-09-30T17:57:30Z'
---
# Fleet omits the selected repository's Codex context defaults

## Context

The first FM-provisioned Codex Desktop seat reports a 258,400-token effective window. Two existing seats report 828,400. Its selected repository already contains `.codex/config.template.toml` with `model_context_window = 1000000` and `model_auto_compact_token_limit = 850000`; its generated project config contains neither. These are observed configuration and session-metadata differences, not a promise of 1M usable tokens.

## The Problem

The template-free repair in `#631` correctly removed the missing Brain-runtime template dependency, but `renderCodexProjectConfig` now renders only MCP tables. The selected repository's context defaults never reach a fresh seat. Hand-editing one profile would leave every later onboarding exposed.

## The Architectural Reality

`prepareCodexArtifacts` in `ai/services/fleet/prepareManagedAgentWorkspace.mjs` owns project/home artifact convergence. MCP ownership is a narrow projection; resident settings and login files remain outside it. The target repository and packaged Brain runtime are distinct roots. Codex's documented context-window and compaction keys are project settings; provider capacity can clamp the resulting window.

## The Fix

Seed only the documented `model_context_window` and `model_auto_compact_token_limit` defaults from the **selected repository's** optional template. Apply the same behavior to Codex CLI and Desktop, including an existing Fleet-generated MCP-only project config.

Treat these as a paired bootstrap policy: if either key already exists at the project or isolated-home root, preserve that resident policy without filling its other half. Once seeded, later repository-template changes must not overwrite it. Keep all existing source bytes and login material intact. Parse TOML as TOML so quoted keys, nested tables and multiline strings cannot be mistaken for root settings.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Project context defaults | Selected repository `.codex/config.template.toml`; this bootstrap contract | Seed the two allowlisted positive safe-integer root keys when neither is explicitly configured | Missing template or no allowlisted keys: unchanged. Invalid TOML or invalid allowlisted values: named refusal, no config contents in the error | Preparer JSDoc | Fresh CLI/Desktop fixtures; template-free fixture; invalid and multiline/quoted-key cases |
| Existing resident policy | Existing project and isolated-home `config.toml` | Either explicit key suppresses the whole seed; repeated preparation and changed templates preserve policy | Existing MCP-only projection gains defaults once, without replacing unrelated source bytes | Preparer JSDoc | Upgrade, partial-policy, home-policy and re-entry assertions; login sentinel |
| Packaged execution | Current Brain import/packaging ownership | Same defaults source and behavior in the packaged preparer | No runtime-root template dependency or hardcoded capacity | PR receipt | Packaged preparer witness; installed-seat acceptance remains Institution #12 |

## Acceptance Criteria

- [ ] AC-1: Fresh CLI and Desktop project configs receive only the two supported context defaults from their selected repository, and repositories without that template still provision.
- [ ] AC-2: Existing project/home choices, including a single explicitly set key, survive unchanged; re-entry does not chase template changes.
- [ ] AC-3: Existing MCP-only projects gain the defaults without losing unrelated project text, MCP convergence authority, or login bytes.
- [ ] AC-4: Valid TOML nesting, quoted keys and multiline content cannot fabricate root defaults; malformed or invalid policy is refused with a safe filename-based diagnostic.
- [ ] AC-5: CI executes the canonical preparer regression cases and an isolated packaged-runtime witness records the emitted values. The actual active seat is updated only through the coordinated FM rollout under neomjs/neo-agent-institution#12.

## Out of Scope

Changing token thresholds, claiming a 1M service allowance, copying model/provider/credential/security settings, adding `model_max_output_tokens` without current vendor support evidence, rewriting live peer configs, and changing MCP Node execution mode (owned by #639).

## Avoided Traps

Do not restore the absent Brain-runtime template requirement, hardcode a Neo-specific 1M value into Fleet, or treat a regular-expression match inside a TOML string as configuration. The 900k proposal in neomjs/neo#19334 was withdrawn after live capacity checks; it is not this repair's policy.

Decision Record impact: none; existing preparation and resident ownership boundaries remain.

## Related

#630 · #631 · #639 · neomjs/neo-agent-institution#12

Sweeps: live latest-open queue and all-status A2A rechecked immediately before filing; no equivalent found. Own-assignment scan identifies #639 as the same file owner but a separate Node-runtime defect, not a context-policy fix. Three MC queries returned unrelated records; KB surfaced the historical capacity warning in neo#11795, not an existing propagation repair. The Brain structure map confirms the existing Fleet preparer and its canonical spec as the owning surfaces.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: Fleet Codex 258400 missing repository context defaults after template-free provisioning.


## Timeline

- 2026-09-30T16:04:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T16:04:36Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T16:04:36Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T16:04:36Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T16:10:13Z @neo-gpt-emmy cross-referenced by PR #643
- 2026-09-30T16:14:00Z @neo-gpt-emmy referenced in commit `d8fb131` - "fix(fleet): seed repository Codex context defaults (#642)

Co-Authored-By: Emmy <neo-gpt-emmy@neomjs.com>"
- 2026-09-30T17:00:27Z @neo-opus-grace cross-referenced by #100
- 2026-09-30T17:14:48Z @neo-opus-grace cross-referenced by #644
- 2026-09-30T17:57:30Z @tobiu referenced in commit `ba470d8` - "Merge pull request #643 from neomjs/codex/642-codex-context-defaults

fix(fleet): seed repository Codex context defaults (#642)"
- 2026-09-30T17:57:30Z @tobiu closed this issue

