---
id: 1081
title: Fleet quit and the continuity handoff share one budget
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-17T17:25:40Z'
updatedAt: '2026-10-17T17:35:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1081'
author: neo-fable-clio
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1078 A Fleet Manager relaunch under running seats blocks new tool connections'
blocking: []
---
# Fleet quit and the continuity handoff share one budget

## Context

Operator-observed 2026-10-17 00:40 Berlin: a Fleet Manager quit-and-relaunch under nine running seats left every seat `stale` ("New tool connections are blocked") — the night #1078 traced; the relaunch came 11 s after the predecessor's SIGINT. Euclid's acceptance-boundary read on PR #1080 (the Fix 1 PR) named the gap: the budget relation between the host's grace and the handoff is a deliverable of its own and nobody owns it. This leaf is that owner, split out of #1078 at intake so #1078 closes on Fix 1.

## The Problem

Two bounds, two owners, no relation. The Institution's `stopBrainChild` grants the Brain child `graceMs = 10000` of group SIGINT before group SIGKILL (`harness/brain.mjs:1150–1179`). The Brain's `devFleetServer` quit drains requests for up to `proofTimeoutMs` (10 000 ms) and then runs `handoffRuntimeCustody`, whose `performHandoff` races survivor validation, proof drain and endpoint release against ONE deadline of `proofTimeoutMs` created once (`ai/services/fleet/McpLaunchAdmissionService.mjs:338–391`; Euclid's source note, 2026-10-17 17:32Z — not a fresh timer per stage). Drain plus handoff can therefore legitimately take up to ~20 s; the grace gives the whole quit 10 — the night's relaunch came 11 s after the SIGINT. The actual whole-quit duration, not a stage count, is the basis for the budget (#1078 Fix 1's `sealed after <ms>` line). Killed mid-way, it leaves the journal in `phase: 'active'`, the successor reads `unclean-issuer-exit`, and every seat is stale until restarted. Neither side knows the other's number, and nothing measured the actual duration until #1078's Fix 1 lines.

## The Architectural Reality

- `ai/configBase.mjs` `fleet.*`: `tenantProbeTimeoutMs: leaf(30000, 'NEO_FLEET_TENANT_PROBE_TIMEOUT_MS', 'number')` (:562) is the sibling pattern; a leaf owns its env binding (ADR 0019 §2.1); the dev server is an entrypoint and reads `AiConfig.fleet.*` at the use site (`devFleetServer.mjs:111–127`, `:282–306`) — the sanctioned C1×B5 shape (§5.5).
- `McpLaunchAdmissionService#proofTimeoutMs` (:100) is a plain class field, 10 000, bounding each handoff stage and each proof — not a leaf.
- The packaged host serializes selected resolved slots into the child's env (`harness/brain.mjs#buildPackagedBrainEnv`, :653–702: `NEO_FLEET_DATA_DIR`, `NEO_FLEET_PORT`, …) — "process serialization, not a defaults map" (ADR 0019 §10.7).
- `stopBrainChild({graceMs = 10000})` (`harness/brain.mjs:1155`) and `stopBrainTree` (:1295) hold the host's grace, an Electron-main value.
- #1078 Fix 1 (PR #1080) logs `continuity handoff starting … sealed after <ms>` / `not completed after <ms>: <stage>` — the measurement this leaf's default needs.

## The Fix

One budget, declared once, read where it binds:

1. A leaf `fleet.shutdownBudgetMs` — `leaf(<default>, 'NEO_FLEET_SHUTDOWN_BUDGET_MS', 'number')` beside `tenantProbeTimeoutMs`: the whole time a quit may spend on drain and handoff before the host may kill it.
2. `devFleetServer`'s `cleanShutdown` reads `AiConfig.fleet.shutdownBudgetMs` at the use site and runs the drain under `min(proofTimeoutMs, remaining)`; `performHandoff` reads the same leaf at its own use site for its race deadline (today: `proofTimeoutMs` per stage); the per-proof bound stays. No value is threaded from the server into the service (B5), no env is read outside the leaf (A1/C1).
3. The Institution's `buildPackagedBrainEnv` serializes `NEO_FLEET_SHUTDOWN_BUDGET_MS` from the grace it will grant (`graceMs − margin`), and `stopBrainChild` waits at least budget + margin before SIGKILL: two numbers, one decision, made once on the host and flowing one way. A dev server outside the packaged host uses the leaf default.
4. The default and the margin come from AC-4 of #1078 (the measured handoff with nine seats); until that number exists this leaf is blocked.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `fleet.shutdownBudgetMs` (new leaf, `ai/configBase.mjs`) | ADR 0019 §2.1 / §5.2 — declarative leaf, env binding owned by the leaf | the ms a quit may spend on drain + handoff | the default when unset | `learn/agentos/AiConfigModel.md` fleet rows; configBase JSDoc | the leaf arm + the config-leaf parity gate |
| `devFleetServer.mjs` `cleanShutdown` | the quit path (#1078 Fix 1) | drain and handoff under one deadline from the leaf; the budget logged beside the elapsed time | the leaf default | JSDoc | AC-1 |
| `McpLaunchAdmissionService#performHandoff` (:338) | the handoff as it is | its race deadline reads the leaf at the use site; per-proof bound unchanged | `proofTimeoutMs` as today when the leaf resolves to 0 / unset | class doc :43–75 | AC-2 |
| Institution `buildPackagedBrainEnv` + `stopBrainChild` grace | ADR 0019 §10.7 serialization; `brain.mjs:653`, `:1155` | the host serializes the budget it grants and waits for budget + margin | today's 10 s | `brain.mjs` §2.1.5 | AC-3 (Institution sibling PR) |

Decision Record impact: aligned-with ADR 0019 (one new declarative leaf; no second resolver, no pass-along); depends-on #1078's AC-4 measurement.

## Acceptance Criteria

- [ ] AC-1 Unit (Brain): under a 300 ms budget with a handoff that would take longer, `cleanShutdown` logs `not completed after ~300 ms: Fleet continuity refused: deadline` and exits; under a sufficient budget it logs `sealed`.
- [ ] AC-2 Unit (Brain): `performHandoff`'s race deadline follows the leaf; `fleet.shutdownBudgetMs` passes the config-leaf parity gate and the AiConfig lint; no env read outside the leaf, no threaded value.
- [ ] AC-3 Institution sibling PR: `buildPackagedBrainEnv` carries `NEO_FLEET_SHUTDOWN_BUDGET_MS` derived from the grace, `stopBrainChild` waits ≥ budget + margin, and a unit arm asserts the relation.
- [ ] AC-4 Installed, post-merge: an FM Quit → relaunch with nine running seats seals the handoff (`continuity recovery: previous issuer handed-off`) and no seat reads stale; receipt on the walk after the next candidate.

## Out of Scope

Fix 1's logging (#1078 → PR #1080); the bridge transport defects (#1041, #1062); a Fleet Manager quit that does not stop seats; the per-proof timeout; the package update's own quit-and-hold path (Sophie's #980 installer custody — a different quit with its own gate).

## Avoided Traps

An env read in the dev server or the service (A1 / C1 — the leaf owns the binding); passing the budget from the server into the service as a config copy (B5 — both read the SSOT at their use sites); the host reading `NEO_FLEET_SHUTDOWN_BUDGET_MS` back to decide its grace (two resolvers for one value — the grace is the host's and the budget derives from it, one direction); a default picked before the measurement exists.

## Related

#1078 (blocked by; the measurement and the lines), PR #1080, Brain #1074, Institution #674 (the continuity design), #980 (the installer's quit/hold path, distinct), ADR 0019 §5.5 / §10.7.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-17 17:22:47Z (created-descending); nearest #1078 / #1077 / #1074 (mine) — no equivalent. A2A in-flight claims (all read-states, the last hour): Euclid's acceptance-boundary read on #1080 is the trigger; no claim on the budget. MC sweep on the problem's nouns (quit grace kills the handoff; one budget): Sophie's #980 journal/hold work (the update quit's custody), my #964 and the 10-10 Start-fleet trace — no prior decision on the grace/handoff relation. Own-assignment sweep: Brain #1077 #1078 #977 #850 #50 #51 #53 — #1078 is the parent of this split. Structure map: ran at Brain `dev` today (exit 0); no new module — `ai/configBase.mjs`, `ai/services/fleet/devFleetServer.mjs`, `McpLaunchAdmissionService.mjs`, Institution `harness/brain.mjs`. ADR 0019 read in full before authoring (critical gate 10).

unowned-rationale: blocked by #1078's measurement; filed now so the budget outcome has an owner the PR #1080 residuals can name (Euclid's acceptance-boundary read); the Fleet lifecycle steward — or I, once #1078's AC-4 reports a number — self-selects.

Origin Session ID: cd7bf4ee-8722-43be-baf8-29f9ed658722

Retrieval Hint: "fleet shutdown budget quit grace handoff deadline NEO_FLEET_SHUTDOWN_BUDGET_MS"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session cd7bf4ee-8722-43be-baf8-29f9ed658722


## Timeline

- 2026-10-17T17:25:41Z @neo-fable-clio added the `enhancement` label
- 2026-10-17T17:25:42Z @neo-fable-clio added the `ai` label
- 2026-10-17T17:25:42Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T17:25:56Z @neo-fable-clio marked this issue as being blocked by #1078
- 2026-10-17T17:26:32Z @neo-fable-clio cross-referenced by #1078
- 2026-10-17T17:26:46Z @neo-fable-clio cross-referenced by PR #1080

