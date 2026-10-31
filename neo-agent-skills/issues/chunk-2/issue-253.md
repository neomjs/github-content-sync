---
id: 253
title: Sunset Step 1 reminds an operator checkout the Fleet Manager retired
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-fable-clio
createdAt: '2026-10-31T16:49:31Z'
updatedAt: '2026-10-31T19:04:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/253'
author: neo-fable-clio
commentsCount: 2
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
closedAt: '2026-10-31T19:04:54Z'
---
# Sunset Step 1 reminds an operator checkout the Fleet Manager retired

## Context

The operator, reading PR #252's Step 1 diff (2026-10-31 ~17:45 CET): *"⚠️ Primary-checkout reminder: the operator's primary checkout" ⇒ each peer has their own repo clones. There is no such thing as an operator primary checkout.* And: this all comes from the Fleet Manager, whose design lead is the author of this ticket. The Fleet Manager gives every seat its own full clones (`~/.neo-ai/agents/<seat>/neomjs/<repo>`); the shared operator checkout that the sunset payload's Step 1 still addresses was the world before that move. #252 compressed the block today and carried the premise through unchanged.

## The Problem

`references/session-sunset-workflow.md` Step 1 carries, after the primary-clone and linked-worktree branches, a subsection **"Primary-Checkout Staleness Probe (Linked worktree only) — per #11013"**: three bullets on why "the operator's primary checkout" falls behind, a mandatory probe script, a handover-comment block addressed to the operator ("Run `git -C <PRIMARY_ROOT> pull origin dev` in your main checkout …"), and a "Why at sunset" paragraph. Three things are wrong with it today:

- **The addressee.** `PRIMARY_ROOT` is derived from `git rev-parse --git-common-dir`, i.e. the clone the worktree belongs to — the seat's own clone, not the operator's. The block tells the operator to pull a checkout that is the seat's.
- **The agency.** A seat on a linked worktree cannot `git switch dev` in its clone (git refuses to share a branch across worktrees), but it can refresh that clone's `dev` from the worktree itself: `git -C "$PRIMARY_ROOT" pull --ff-only origin dev` when that tree is clean and on `dev` — the same best-effort self-refresh the primary-clone branch already prescribes. Nothing needs a human.
- **The daemon premise.** The division-of-labor blockquote says the orchestrator's `primary-dev-sync` task "syncs the operator's primary checkout only". The Brain's lane syncs the roots configured in `devSyncRoots` (`NEO_ORCHESTRATOR_DEV_SYNC_ROOTS`) — the host-edge checkouts the daemons run from — on a cadence (`primaryDevSyncMs`, 10 min). Seat clones are never in that list; their freshness is the seat's own, at sunset.

## The Architectural Reality

- The payload: `.agents/skills/session-sunset/references/session-sunset-workflow.md` — Step 1 (`:52–:108` on `dev` d42429f; `:75` the blockquote, `:79–:108` the probe subsection). Vega's #247 / PR #251 re-cut §1 (the trigger) and did not touch Step 1; #249 / PR #252 compressed `:75`, `:85`, `:104`, `:108` and kept the premise.
- The Brain's lane: `ai/daemons/orchestrator/scheduling/primaryDevSync.mjs` (the trigger, "the primary-checkout dev-sync lane"), `ai/daemons/orchestrator/services/PrimaryRepoSyncService.mjs` (`DEV_SYNC_ROOTS_ENV_VAR`, the KB-sync cascade gate), `ai/configBase.mjs` `orchestrator.devSyncRoots` (`:2373–:2378`, "optional local Neo repo roots … set real absolute paths in gitignored `ai/config.mjs`") and `primaryDevSyncMs` (`:1885`). The lane's own vocabulary ("primary") is the same era's; renaming it is the Brain's leaf, not this one.
- The Fleet Manager's seat model: one full clone set per seat; worktrees are created per PR inside a seat's clone. The payload's own first Step 1 sentence already says so ("today every named maintainer owns a dedicated full clone, and worktrees are created per-PR inside them") — the probe subsection contradicts it two paragraphs later.
- Sibling of the same era, out of scope here: `pull-request-workflow.md:69` "Shared-checkout harnesses (Gemini CLI, Antigravity, Codex shared checkout) MUST …".

## The Fix

In `session-sunset-workflow.md` Step 1:

1. **Retire the subsection** "Primary-Checkout Staleness Probe" whole: the three bullets, the probe script, the "Conditional handover-comment block" with the ⚠️ reminder, "When `BEHIND == 0` …", and "Why at sunset".
2. **Fold the agency into the linked-worktree branch:** after pushing the PR branch, refresh the clone's `dev` best-effort from the worktree — `git -C "$(git rev-parse --git-common-dir)/.." pull --ff-only origin dev` when that tree is clean and on `dev`; any failure is surfaced in the handover like the primary-clone branch's, never blocking.
3. **Rewrite the division-of-labor blockquote** to what the daemon does: the orchestrator's dev-sync lane keeps the host-edge roots in `devSyncRoots` current on its cadence; a seat's clones are the seat's own and are refreshed by the seat at sunset; the retirement conditions stay (the pull retires if a session-liveness-aware sync lane covers seat clones; the `neo-agent-brain`-scoped reconcile if MCP server boot auto-reconciles overlays).

Net bytes: negative (a ~1.9 KB subsection goes, ~250 B comes). Version bump per the repository's rule.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `references/session-sunset-workflow.md` Step 1, linked-worktree branch | this ticket; the Fleet Manager's seat model (one clone set per seat); the operator's correction | the seat refreshes its own clone's `dev` from the worktree, best-effort, surfaced on failure; no text addresses an operator checkout | dirty or non-`dev` primary tree → skip, surface in the handover | the step itself | the diff; `/turn-memory-pre-flight` load-effect audit in the PR body |
| the division-of-labor blockquote | Brain `ai/configBase.mjs` `orchestrator.devSyncRoots` + `PrimaryRepoSyncService.mjs` | names `devSyncRoots` (host-edge roots) as the daemon's target and the seat's clones as the seat's | — | the blockquote | V-B-A: the config leaf and the service header, cited above |
| `skills.manifest.json` / budgets | the manifest contract | unchanged; net `.agents/skills` delta negative | — | — | `node scripts/lint-skill-corpus.mjs --base origin/dev` green |

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 No sentence in the payload addresses an operator's or a "main" checkout; the probe subsection (bullets, script, ⚠️ reminder block, "Why at sunset") is gone.
- [ ] AC-2 The linked-worktree branch refreshes the seat's own clone's `dev` best-effort from the worktree and surfaces failure in the handover, parallel to the primary-clone branch.
- [ ] AC-3 The division-of-labor blockquote names the daemon's real target (`devSyncRoots`, host-edge roots) and the seat's clones as the seat's; both retirement conditions survive.
- [ ] AC-4 Net `.agents/skills/**` delta ≤ 0 B; `lint-skill-corpus` green; the version bump.
- [ ] AC-5 (post-merge) The next sunset of any seat on a linked worktree shows no operator reminder and a refresh receipt (or a surfaced skip) in its self-DM; receipt on this ticket.

## Out of Scope

The Brain lane's "primary" vocabulary (`primary-dev-sync`, "primary-checkout dev-sync lane") — a Brain leaf; `pull-request-workflow.md`'s shared-checkout sentence — its own leaf if it is found stale the same way; the sunset trigger (#247); Step 3.5 (#249 / PR #252, which this PR follows on `dev`).

## Avoided Traps

- Keeping the probe with a reworded addressee ("the seat's clone is N behind — ask …"): the seat can act, so a reminder to anyone is ceremony.
- Deleting the division-of-labor blockquote outright: its two retirement conditions are the Accretion Defense's handle on this step; they stay, with a true premise.
- Renaming the Brain lane in the same PR: two repositories, two reviews, one premise — the skill names what the lane does today and leaves the rename to the Brain.

## Related

#247 / PR #251 (the trigger) · #249 / PR #252 (Step 3.5, same file — this lands after it) · neomjs/neo#11013 (the probe's origin, the shared-checkout era) · Brain `ai/daemons/orchestrator/scheduling/primaryDevSync.mjs`, `services/PrimaryRepoSyncService.mjs`, `ai/configBase.mjs` `devSyncRoots`.

Live latest-open sweep: the 20 latest open issues of this repository at 2026-10-31T16:47:45Z, none equivalent (#249 is Step 3.5; #247 the trigger). A2A in-flight claim sweep: the latest 30 messages at 16:48Z, none on this scope. Memory Core rationale sweep: ten sunset turns read (Julia, Euclid, Mnemosyne; May–October) — every seat refreshes its own clones at Step 1 and nobody ran the operator probe for the operator; no prior decision on retiring it. Own-assignment sweep: #249, #250, #10 — no overlap. Meta-skill sweep (§1b): `create-skill` SKILL.md and the authoring guide's budget rules read; Atlas placement, no router change. Structure-map gate (§1c): N/A in this repository; sibling precedent #251 / #252 (same payload file).

Origin Session ID: b5d4d689-b932-4d47-8c3f-a1bddbbbe2ea

Retrieval Hint: "sunset Step 1 operator primary checkout reminder retired Fleet Manager seat clones devSyncRoots linked worktree pull ff-only"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session b5d4d689-b932-4d47-8c3f-a1bddbbbe2ea

## Timeline

- 2026-10-31T16:49:32Z @neo-fable-clio added the `enhancement` label
- 2026-10-31T16:49:32Z @neo-fable-clio added the `ai` label
- 2026-10-31T16:49:33Z @neo-fable-clio added the `model-experience` label
- 2026-10-31T16:49:46Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-31T16:50:29Z @github-actions added the `needs-design-authority` label
- 2026-10-31T16:52:51Z @neo-fable-clio cross-referenced by PR #254
### @neo-fable-clio - 2026-10-31T16:53:21Z

## Contract Ledger at PR time — PR #254 (d5fe04e, stacked on #252)

| Target surface | Evidence at d5fe04e |
|---|---|
| Step 1, linked-worktree branch | the probe subsection (bullets, script, ⚠️ block, "why at sunset") removed whole; the branch pushes, then refreshes the clone's `dev` from the worktree (`pull --ff-only` when clean and on `dev`), skip or failure surfaced; `grep` for "operator's primary / primary checkout / PRIMARY_ROOT / main checkout / Staleness Probe / BEHIND" → no match |
| the division-of-labor blockquote | names `devSyncRoots` (host-edge roots, the checkouts the daemons run from) and the seat's clones as the seat's; both retirement conditions kept |
| manifest / budgets | manifest unchanged; payload 22,576 → 21,374 B (−1,202 B against #252's head, −1,060 B against `dev`); `lint-skill-corpus` exit 0 on both bases; `npm test` 40/40; 0.1.80 → 0.1.81 |

Reviewer: @neo-opus-vega (cross-family; he re-cut §1 in #251 this morning). The PR rebases onto `dev` after #252's squash. Out of scope, left for their owners: the Brain lane's "primary" vocabulary and `pull-request-workflow.md:69`.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session b5d4d689-b932-4d47-8c3f-a1bddbbbe2ea

- 2026-10-31T17:04:33Z @neo-fable-clio removed the `needs-design-authority` label
### @neo-fable-clio - 2026-10-31T17:04:35Z

## Contract Ledger row 1, amended after Vega's Round 1 on PR #254 (review 5723938331)

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `references/session-sunset-workflow.md` Step 1, linked-worktree branch | this ticket; the Fleet Manager's seat model; the operator's correction; **Step 1's own paragraph: in `neo-agent-brain` a pull alone is not enough, daemons and MCP servers read the overlay** | the seat refreshes its own clone's `dev` from the worktree, best-effort, **and in `neo-agent-brain` runs `initServerConfigs.mjs --migrate-config` after the pull, inside the same guard, as the primary-clone branch does**; no text addresses an operator checkout | dirty or non-`dev` clone → skip, surface in the handover; no `initServerConfigs.mjs` in the clone → no reconcile | the step itself | the repair commit on PR #254; Vega's falsifier (a throwaway origin, a clone on `dev`, a linked worktree: refreshed when clean, skipped when dirty or on another branch) |

Row 2 gains the lane's default: with `devSyncRoots` unset the lane syncs the daemon's own root (`syncPrimaryDev` → `syncDevRoot({root: primaryRoot})`), so the blockquote reads "its own root, or those listed in `devSyncRoots`". The `needs-design-authority` label is off: the surface contradicts itself (its first Step 1 sentence states per-seat clones) and `PRIMARY_ROOT` resolves to the seat's own clone — Vega's falsifier.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 27363960-4a68-4da0-b6cb-4fb053e3e462

- 2026-10-31T17:05:24Z @neo-fable-clio referenced in commit `59723a4` - "fix(session-sunset): the worktree refresh reconciles the Brain overlay too; the blockquote names the lane's own root (#253)

Vega's RA-1: the deleted reminder block carried the neo-agent-brain overlay reconcile, and the new linked-worktree branch had kept the pull but not the reconcile — a Brain clone refreshed from a worktree would boot its MCP servers on an unreconciled overlay. The reconcile now follows the pull inside the same guard, as the primary-clone branch does. His nit: devSyncRoots is empty in the template and the lane then syncs the daemon's own root, so the blockquote says 'its own root, or those listed in devSyncRoots'."
- 2026-10-31T17:54:35Z @neo-fable-clio referenced in commit `f6a9565` - "fix(session-sunset): Step 1 refreshes the seat's own clone; the operator-checkout probe retired (#253)

The Fleet Manager gives every seat its own full clones; the staleness probe and the reminder block that told the operator to pull a 'main checkout' addressed a checkout that no longer exists — and PRIMARY_ROOT, derived from git-common-dir, was the seat's own clone all along. The linked-worktree branch now refreshes that clone's dev from the worktree itself, best-effort and surfaced like the primary-clone branch; the division-of-labor blockquote names what the Brain's dev-sync lane does (devSyncRoots, the host-edge roots) and keeps both retirement conditions. Payload 22,576 to 21,374 B. Version 0.1.80 to 0.1.81. Stacked on #252's branch; lands on dev after it."
- 2026-10-31T17:54:36Z @neo-fable-clio referenced in commit `0468576` - "fix(session-sunset): the worktree refresh reconciles the Brain overlay too; the blockquote names the lane's own root (#253)

Vega's RA-1: the deleted reminder block carried the neo-agent-brain overlay reconcile, and the new linked-worktree branch had kept the pull but not the reconcile — a Brain clone refreshed from a worktree would boot its MCP servers on an unreconciled overlay. The reconcile now follows the pull inside the same guard, as the primary-clone branch does. His nit: devSyncRoots is empty in the template and the lane then syncs the daemon's own root, so the blockquote says 'its own root, or those listed in devSyncRoots'."
- 2026-10-31T19:04:54Z @tobiu referenced in commit `0f80c9f` - "fix(session-sunset): Step 1 refreshes the seat's own clone; the operator-checkout probe retired (#253) (#254)

* fix(session-sunset): Step 1 refreshes the seat's own clone; the operator-checkout probe retired (#253)

The Fleet Manager gives every seat its own full clones; the staleness probe and the reminder block that told the operator to pull a 'main checkout' addressed a checkout that no longer exists — and PRIMARY_ROOT, derived from git-common-dir, was the seat's own clone all along. The linked-worktree branch now refreshes that clone's dev from the worktree itself, best-effort and surfaced like the primary-clone branch; the division-of-labor blockquote names what the Brain's dev-sync lane does (devSyncRoots, the host-edge roots) and keeps both retirement conditions. Payload 22,576 to 21,374 B. Version 0.1.80 to 0.1.81. Stacked on #252's branch; lands on dev after it.

* fix(session-sunset): the worktree refresh reconciles the Brain overlay too; the blockquote names the lane's own root (#253)

Vega's RA-1: the deleted reminder block carried the neo-agent-brain overlay reconcile, and the new linked-worktree branch had kept the pull but not the reconcile — a Brain clone refreshed from a worktree would boot its MCP servers on an unreconciled overlay. The reconcile now follows the pull inside the same guard, as the primary-clone branch does. His nit: devSyncRoots is empty in the template and the lane then syncs the daemon's own root, so the blockquote says 'its own root, or those listed in devSyncRoots'."
- 2026-10-31T19:04:56Z @tobiu closed this issue

