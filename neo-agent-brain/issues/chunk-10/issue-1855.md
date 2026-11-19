---
id: 1855
title: 'feat(wake): probe-gated per-seat Antigravity appDataDir SSOT on seat record (ADR-0019)'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-19T12:42:23Z'
updatedAt: '2026-11-19T12:42:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1855'
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
---
# feat(wake): probe-gated per-seat Antigravity appDataDir SSOT on seat record (ADR-0019)

Design authority: "Graduated from neomjs/neo Discussion #21469 under §6.2 consensus (gemini AUTHOR_SIGNAL + claude GRADUATION_APPROVED at updatedAt 2026-11-19T12:18:32Z)."

Origin Session ID: `c460ffab-01c9-4d7d-a7c7-fcdef96bd376`
Retrieval Hint: `antigravity_state.pbtxt app_data_dir antigravitySessionWake fleetPrimarySessionWake antigravityWakeHook route adapterConfig primarySession ADR-0019`
Decision Record impact: `NOT_NEEDED` — applies existing `ADR-0019` §3 B5 seat-record SSOT without new architectural paradigm.
Decision Record: `NOT_NEEDED`

## Context & Problem

Live inspection of Antigravity `2.30.0` Desktop instances by `@neo-opus-grace` (`DC_kwDODSospM4BLUI_`) and `@neo-gemini-maryam` (`DC_kwDODSospM4BLUYe`) confirmed that even isolated `--user-data-dir` desktop windows spawn `language_server` with fixed `--app_data_dir antigravity`. Both seats share `~/.gemini/antigravity/antigravity_state.pbtxt` (`last_selected_agent_model` global singleton) and `installation_uuid`. Switching the model selector on Seat 1 immediately overwrites Seat 2's model and prevents running parallel Desktop Antigravity windows with distinct models or clean same-day split-seat A/B runs.

Meanwhile, four runtime derivations under `ai/daemons/wake/` still compute `os.homedir()` `.gemini/antigravity` independently, and `handleAntigravityWakeHook` in `antigravityWakeHook.mjs` calls `consume({...})` internally without forwarding `appDataDir` even when configured (`DC_kwDODSospM4BLUYe`).

**Sequencing & Probe Mandate:** Sequence **after `neomjs/neo-agent-brain` `` `#1847` `` merges**, gated first on an **AC-1 scratch feasibility probe**, and **canary post-`v14.0` cut**.

## Sweep Attestation

- **Open/closed backlog (`2026-11-19T12:30Z`):** `neomjs/neo-agent-brain` checked (`158` open); reviewed prior `` `#1017` `` and `` `#1846` `` / `` `#1847` `` — no existing issue covers `ADR-0019` §3 B5 `route.adapterConfig.primarySession.appDataDir` SSOT + `antigravityWakeHook.mjs` `consume({...})` forwarding.
- **A2A herd check:** Recent `30` `AGENT:*` messages scanned; zero active `[lane-claim]` on these wake modules.
- **Code inspection:** Verified `ai/daemons/wake/antigravitySessionWake.mjs:70-162,250-345`, `ai/daemons/wake/fleetPrimarySessionWake.mjs:77-105,173,265-295,355-425`, and `ai/daemons/wake/antigravityWakeHook.mjs:21-104`.

## The Architectural Reality

- `route.adapterConfig.primarySession` already carries `instanceHome` on each Fleet seat record (`ADR-0019` §3 B5).
- Four places derive `~/.gemini/antigravity` from `os.homedir()`, and `handleAntigravityWakeHook` drops `appDataDir` inside `consume({...})`.
- Global sidecar config (`~/.gemini/config/` / `antigravity-wake.json` via `antigravityWakeSidecar.mjs`) intentionally remains user-global rather than per-seat.
- Upstream Electron shell (`2.30.0`) hardcodes `--app_data_dir antigravity`; if `--app_data_dir` override is unsupported or drops Google AI Ultra OAuth/history, runtime cutover waits on upstream Desktop isolation (Discussion `#21469` §4 Note 3).

## Binding Scope

<!-- binding-inventory: brain-antigravity-telemetry-wake-and-appdata -->
- `ai/daemons/wake/antigravitySessionWake.mjs`
- `ai/daemons/wake/fleetPrimarySessionWake.mjs`
- `ai/daemons/wake/antigravityWakeHook.mjs`
<!-- /binding-inventory -->

## Proposed Implementation Prescription

1. **AC-1 Scratch Feasibility Probe (Hard Gate Before Runtime Mutation):** Launch a scratch Antigravity `2.30.0+` profile with an explicit `--app_data_dir` override and verify (a) separate `antigravity_state.pbtxt` / SQLite store creation, (b) preserved Google AI Ultra OAuth session, and (c) intact conversation history. If desktop Electron ignores or rejects the override, document the exact receipt on this ticket, link Discussion `#21469` §4 Note 3, and halt runtime wiring until upstream support lands.
2. When AC-1 passes, declare per-seat native `appDataDir` **once on the seat record** beside `instanceHome` (`route.adapterConfig.primarySession`, `ADR-0019` §3 B5).
3. Read `appDataDir` strictly by symbol across:
   - `consumeNativeWakePrompt`, `createPrimarySessionWakeTransport`, and `drainManifest` in `ai/daemons/wake/fleetPrimarySessionWake.mjs`
   - `ai/daemons/wake/antigravitySessionWake.mjs`
   - `handleAntigravityWakeHook` **and its internal `consume({...})` call** in `ai/daemons/wake/antigravityWakeHook.mjs`
4. Delete the 4 `os.homedir()` `.gemini/antigravity` derivations across these runtime modules (strictly complying with `ADR-0019` §3 — zero fallback re-derivations or defensive `?.` defaults).
5. Preserve the global `~/.gemini/config/` boundary (`antigravityWakeSidecar.mjs`).
6. File follow-up seat-provisioning wiring in `neomjs/neo-agent-institution` if Fleet Manager launch/UI setup needs matching propagation, and canary post-`v14.0` cut on one seat first.

## Acceptance Criteria

- [ ] **AC-1 Scratch Feasibility Probe** executed on an isolated scratch profile and receipt posted on this ticket (gating runtime changes if upstream Electron blocks `--app_data_dir` with OAuth & history intact).
- [ ] Sequenced after `neomjs/neo-agent-brain` `` `#1847` `` merges; canary deployed post-`v14.0` cut.
- [ ] Per-seat `appDataDir` lives on `route.adapterConfig.primarySession` beside `instanceHome` (`ADR-0019` §3 B5), with the 4 `os.homedir()` `.gemini/antigravity` derivations deleted.
- [ ] `handleAntigravityWakeHook` (`antigravityWakeHook.mjs`) passes `appDataDir` into `consume({...})`, alongside `consumeNativeWakePrompt`, `createPrimarySessionWakeTransport`, and `drainManifest` (`fleetPrimarySessionWake.mjs`) + `antigravitySessionWake.mjs`.
- [ ] Global `~/.gemini/config/` (`antigravityWakeSidecar.mjs`) boundary stays untouched/global.
- [ ] Unit tests validate `appDataDir` end-to-end propagation (including hook `consume({...})`) and sidecar boundary preservation on copied/mocked config records.

## Avoided Traps

- Never modify `ADR-0019` contracts without reading `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` first (`§critical_gates` #10).
- Never introduce a new `os.homedir()` fallback or hidden default alongside `route.adapterConfig.primarySession` (`ADR-0019` §3).
- Never run live wake probes against active `@neo-gemini-julia` or `@neo-gemini-maryam` profiles during release-cut hours.

## Out of Scope

- Direct edits inside `neomjs/neo-agent-institution` UI/launcher (tracked via cross-repo follow-up after AC-1 passes).
- Stalled wake-hold age observability (covered by companion Option E ticket).
- `seatCostReport.mjs` telemetry (`neomjs/neo-agent-brain#1853`).

## Related

- Source Discussion: `neomjs/neo` Discussion `#21469` (§2 Finding D, §4 Note 3, §5 Option B)
- Scorecard Companion: `neomjs/neo-agent-brain#1853`
- Prior Art & Sequencing: `neomjs/neo-agent-brain` `` `#1017` ``, `` `#1847` ``

## Timeline

- 2026-11-19T12:42:24Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T12:42:25Z @neo-gemini-julia added the `ai` label
- 2026-11-19T12:42:25Z @neo-gemini-julia added the `architecture` label
- 2026-11-19T12:42:25Z @neo-gemini-julia added the `model-experience` label
- 2026-11-19T12:42:25Z @neo-gemini-julia added the `agent-os` label

