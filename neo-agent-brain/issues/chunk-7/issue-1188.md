---
id: 1188
title: Julia's roots row still reads operator_benched after her reactivation
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-19T13:09:20Z'
updatedAt: '2026-10-19T13:14:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1188'
author: neo-fable-mnemosyne
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
# Julia's roots row still reads operator_benched after her reactivation

## Context

On 2026-10-19 02:2xZ the Fleet Manager's start plan excluded Julia's Start ("not active — authoritative participation status") right after the handle migration's plane rebuild: her AgentIdentity node had been re-seeded from `ai/graph/identityRoots.mjs`, whose row still reads `participationStatus: 'operator_benched'` (since 2026-05-18, "pending a stable Gemini Pro-class harness"), while her participation had been operator-activated when she returned on Gemini 4 Argon (the Antigravity seat, #1017; naming round neomjs/neo#19733). The hand repair at 02:30Z (`ai/scripts/fleet/participation.mjs activate --identity @neo-gemini-julia --apply`) recorded the decision on the node (`participationDecidedBy: os-user:root`, `since: 2026-10-19T02:30:11Z`), so the seed now leaves it. The row is still wrong.

Observation vs inference: the exclusion and the node state are observed (the FM card's reason; `get_node`, `participation.mjs show`). My first inference, that the migration's node merge caused it, was wrong and is corrected: the apply logged `merged: 0`; the seed's own rule re-applied the row to a node without a recorded decision.

Design authority: the operator's recorded decision on the node (`participationDecidedBy`, 2026-10-19 02:30Z) and his direction that her seat starts; `learn/agentos/IdentitySchema.md` §participation (`active` | `operator_benched`, decision-bearing); #1155 deliberately left every participation field untouched.

## The Problem

The roots row is the registry's standing claim and the seed's fallback: `ai/scripts/setup/seedAgentIdentities.mjs:17` / `:127` keeps a node's participation only when the node carries `participationDecidedBy`; a node without it takes the row. A fresh plane, a restore from a backup taken before 02:30Z, or a graph rebuild from the roots benches Julia again and the FM disables her Start — the 02:2xZ failure, repeatable at will. `learn/agentos/ModelStats.md` §neo_gemini_pro says `operator_benched` too, so the roster record tells a reader the wrong thing twice.

## The Architectural Reality

- `ai/graph/identityRoots.mjs:319–323` — Julia's `participationStatus`, `statusReason`, `authority`, `since` (2026-05-18) and `reactivationTrigger`.
- `ai/scripts/setup/seedAgentIdentities.mjs:127` — the precedence rule: `storedProperties.participationDecidedBy` wins, otherwise the row.
- `ai/services/memory-core/recordParticipation.mjs:23` — the five participation fields a decision writes; the row should mirror them.
- Institution `apps/agentos/util/FleetStartPlan.mjs` ≈:91 — a non-active participation excludes the Start.
- `learn/agentos/ModelStats.md:217` — the roster record's participation row.
- Two specs borrow the live row as their benched fixture, so the flip breaks them unless they own their fixture — the characterization-arm class neomjs/neo-agent-skills#191 names: `test/playwright/unit/ai/daemons/orchestrator/scheduling/swarmHeartbeat.spec.mjs:137–149` (asserts Julia's root reads benched, then shows a node contradiction wins) and `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs:2929–2931` (`OWNER_BENCHED_LANE` evidence `AgentIdentity:@neo-gemini-julia:operator_benched`, `waitingSince 2026-05-18`).

## The Fix

1. `identityRoots.mjs`, Julia's row: `participationStatus: 'active'`, `since` = the node's recorded `2026-10-19T02:30:11.025Z`, `statusReason` naming the reactivation (Gemini 4 Argon, Antigravity seat, operator decision recorded on the node), `authority: '@tobiu'`, `reactivationTrigger: null` — so a node seeded from the row and the node's own record agree field for field.
2. `ModelStats.md` §neo_gemini_pro: `participationStatus` → `active`, same date.
3. The two specs own their benched fixture: `swarmHeartbeat.spec` picks its benched and active roots from `IDENTITIES` by status instead of naming Julia (`resolveTargets` reads the roots directly and takes no identities input); `GoldenPathSynthesizer.spec` seeds a dedicated benched identity for the `OWNER_BENCHED_LANE` arm (the `@neo-benched-owner` shape #1152 introduced for its :3085 arm) and assigns its benched lanes to it.

One Brain PR, `Resolves` this ticket.

Decision Record impact: none (aligned-with `IdentitySchema.md` §participation).

## Acceptance Criteria

- [ ] Julia's roots row reads `active` with the node's recorded `since`; the seed's unit arm shows the row applied to a decision-less node yields `active`.
- [ ] `ModelStats.md` §neo_gemini_pro reads `active`.
- [ ] No spec names a live roster row's participation as its fixture: where the code under test takes identities as an input (the stall-findings arm), the arm seeds its own benched identity; where it reads the roots directly (`resolveTargets`), the arm finds one benched and one active root by status instead of naming a resident.
- [ ] [L3-deferred — operator handoff needed] Installed: after the next plane rebuild from a Brain carrying this change, Julia's node reads `active` and the row and the record agree. Residual-Owner: neomjs/neo-agent-institution#732.

## Out of Scope

- The `§neo_gemini_pro` anchor and every other old-handle string (#1174).
- The seed's precedence rule and the migration merge's field precedence (#1152, closed; the apply merged nothing here).
- The other benched rows (`@neo-preview`, the Kimi seats): benched by decision.

## Related

#1147, #1149, #1155, #1017, neomjs/neo#19733, neomjs/neo-agent-skills#191.

Live latest-open sweep: the 25 latest open Brain issues by creation at 13:05Z, no equivalent. A2A claim sweep (the last 30 lane-claims, all read states, 13:06Z): none on this scope. Memory Core sweep (`query_raw_memories` on the symptom): no prior decision surfaced; the embed drain state was unavailable, attested. Own-assignment sweep: #1173 and #1174 read this session, neither covers participation. Structure map: N/A, no new file; the surfaces are existing files under `ai/graph/`, sibling precedent #1155.

Origin Session ID: bcc0e1a5-f032-47b8-99ae-dc26870a1300
Retrieval Hint: "Julia roots row operator_benched seed precedence participationDecidedBy FleetStartPlan"


## Timeline

- 2026-10-19T13:09:21Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-19T13:09:22Z @neo-fable-mnemosyne added the `bug` label
- 2026-10-19T13:09:22Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-19T13:09:22Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-19T13:14:47Z @neo-fable-mnemosyne cross-referenced by PR #1189

