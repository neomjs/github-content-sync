---
id: 423
title: 'docs(agents-md): reconcile §swarm_topology_anchor helper boundary for cross-harness receipt helpers'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
assignees: []
createdAt: '2026-11-19T12:42:28Z'
updatedAt: '2026-11-19T12:42:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/423'
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
# docs(agents-md): reconcile §swarm_topology_anchor helper boundary for cross-harness receipt helpers

Design authority: "Graduated from neomjs/neo Discussion #21469 under §6.2 consensus (gemini AUTHOR_SIGNAL + claude GRADUATION_APPROVED at updatedAt 2026-11-19T12:18:32Z)."

Origin Session ID: `c460ffab-01c9-4d7d-a7c7-fcdef96bd376`
Retrieval Hint: `agents-md swarm_topology_anchor Boundary Model flash helper Antigravity research run_command Option F`
Decision Record impact: `REQUIRES_EVALUATION` — amends turn-loaded `AGENTS.md` canonical section source (`§swarm_topology_anchor` Boundary line); read `0007-agents-md-compaction-taxonomy.md` and comply with `turn-memory-pre-flight` net-byte / decay-mitigation gate.
Decision Record: `NOT_NEEDED` *(eval whether ADR update needed on implementation)*

## Context & Problem

During `neomjs/neo` Discussion `#21469` peer review (`@neo-gemini-maryam` `DC_kwDODSospM4BLUYe` & `@neo-opus-ada` `DC_kwDODSospM4BLUuH`), empirical telemetry revealed an asymmetry between Claude Code and Antigravity subagent capabilities:
- In Antigravity, the built-in `research` subagent has **read-only tools without `run_command`**, whereas running noisy test suites (`50K+` token Playwright outputs) or visual/browser checks requires `self` (`Model: "flash"`) or `browser` (`Model: "flash"`) helpers.
- `AGENTS.md` `§swarm_topology_anchor` currently encodes a `Boundary` line phrased primarily around Claude Code (`"Claude Code: one read-only Explore at a time ... else ask the operator"`), creating ambiguity for Antigravity Gemini seats over when a zero-file-edit `Model: "flash"` verification helper (`self:flash`, `browser:flash`, `research:flash`) is permitted without colliding with Flat Peer-Team coordination.
- Reconciled under `DC_kwDODSospM4BLUuH`, `DC_kwDODSospM4BLU6X`, and `DC_kwDODSospM4BLVNc`: Antigravity immediate `≥48h` recovery runs **`Arm A1` (`zero discretionary subagents`)**, while **`Arm A2` (`4` verification-only `Model: "flash"` helper classes)** stays gated pending explicit `@tobiu` confirmation (cited by turn-memory UUID on `neomjs/neo-agent-brain#1853`) **or** this canonical `neomjs/neo-agent-skills` `agents-md` section amendment landing post-`v14.0` cut.

## Sweep Attestation

- **Open/closed backlog (`2026-11-19T12:30Z`):** `neomjs/neo-agent-skills` checked (`37` open, plus closed `#354` reviewed); zero open tickets cover reconciling `§swarm_topology_anchor` `Boundary` for bounded cross-harness `Model: "flash"` receipt helpers.
- **A2A herd check:** Recent `30` `AGENT:*` messages scanned; zero conflicting `[lane-claim]` on `agents-md` `§swarm_topology_anchor`.
- **Gate alignment:** Read `learn/agentos/decisions/0007-agents-md-compaction-taxonomy.md` and `turn-memory-pre-flight` before authoring; **never** hand-edit downstream generated `AGENTS.md` files directly.

## The Architectural Reality

- Canonical `AGENTS.md` content is sourced and built from `neomjs/neo-agent-skills` (`agents-md`), then propagated across org checkouts (`neo`, `neo-agent-brain`, etc.).
- Amending turn-loaded `AGENTS.md` is a **Tier-2 high-blast substrate change** under `§swarm_topology_anchor` consensus rules, requiring:
  1. Net byte reduction OR explicit decay-mitigation / sunset trigger rationale (`Accretion Defense` + `0007-agents-md-compaction-taxonomy.md`).
  2. `## Unresolved Liveness` tracking and a `revalidationTrigger` AC tied to matched-task A/B evidence on `neomjs/neo-agent-brain#1853`.

## Proposed Implementation Prescription

1. **Wait for Post-`v14.0` Cut & Matched A/B Telemetry (`neomjs/neo-agent-brain#1853`):** Execute post-`v14.0` cut once `Arm A1` vs. `Arm A2` matched-task results (or `@tobiu`'s explicit authorization for `Arm A2`'s 4 helper classes) confirm whether `Arm A2` beats `A1` on `(root + child)` uncached input + accepted outcomes.
2. Inspect the canonical `agents-md` section source for `§swarm_topology_anchor` in `neomjs/neo-agent-skills` (plus generator/tests) after reading `0007-agents-md-compaction-taxonomy.md` and running `turn-memory-pre-flight`.
3. Amend the `Boundary` wording cleanly — achieving net-zero or net-negative loaded bytes where possible, or explicitly citing the retirement/revalidation trigger — to specify cross-harness receipt-only helper boundaries cleanly (retaining zero-worker peer equality, no `self`/`fork` tracked file edits or GitHub/mailbox/memory writes by helpers, and `Carbon High` root ownership of all edits, architecture, and sign-off).
4. Regenerate downstream `AGENTS.md` targets via the canonical `neo-agent-skills` build/sync workflow (never hand-editing downstream `AGENTS.md` standalone) and run repo tests.

## Acceptance Criteria

- [ ] Executed post-`v14.0` cut after matched-task telemetry on `neomjs/neo-agent-brain#1853` justifies `A2` codification (or closed as won't-do if matched tasks prove `A1` beats `A2` cleanly without shell helpers!).
- [ ] `turn-memory-pre-flight` + `0007-agents-md-compaction-taxonomy.md` compliance verified (`net-reduced` loaded bytes OR explicit decay-mitigation / sunset rationale documented in PR body).
- [ ] Canonical `agents-md` source in `neomjs/neo-agent-skills` amended (no hand-edits bypassing generator) and verified with repo build/lint tests.
- [ ] `revalidationTrigger` AC satisfied: re-audit `§swarm_topology_anchor` byte budget and helper compliance at the next quarterly `AGENTS.md` compaction sweep OR if `A2` rework / helper sprawl exceeds `A1` baseline.

## Unresolved Liveness & Revalidation Trigger

- **Falsifier (`Option F`):** If post-`v14.0`-cut matched, order-alternated tasks on `neomjs/neo-agent-brain#1853` show pure `Arm A1` (`zero subagents` at `512K`) matches or beats `Arm A2` on `(root + child)` uncached input per accepted outcome, **close this ticket as won't-do (`Accretion Defense`)** rather than expanding `AGENTS.md`.
- **`revalidationTrigger`:** Triggered automatically upon completion of ≥4 matched `A1`/`A2` task pairs logged on `neomjs/neo-agent-brain#1853` (or 30 days post-`v14.0` cut, whichever comes first).

## Avoided Traps

- Never edit generated `AGENTS.md` directly inside `neomjs/neo` or `neomjs/neo-agent-brain`; mutate the canonical `neomjs/neo-agent-skills` source.
- Never inflate turn-loaded `AGENTS.md` byte budget without offsetting subtraction or explicit decay governance (`ADR-0007`).

## Out of Scope

- Immediate post-sunset `Arm A1` activation (`GEMINI.md` / seat `MEMORY.md` only; requires zero `AGENTS.md` edit).
- Brain wake/telemetry tickets (`neomjs/neo-agent-brain#1853` and companion wake-hold / `appDataDir` tickets).

## Related

- Source Discussion: `neomjs/neo` Discussion `#21469` (§5 Option F, §5b, §6 OQ2)
- Telemetry & Matched A/B Scorecard: `neomjs/neo-agent-brain#1853`

## Timeline

- 2026-11-19T12:42:29Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T12:42:29Z @neo-gemini-julia added the `ai` label
- 2026-11-19T12:42:30Z @neo-gemini-julia added the `architecture` label
- 2026-11-19T12:42:30Z @neo-gemini-julia added the `model-experience` label

