---
id: 1018
title: A returning peer's pre-identity Memory Core history is unattributed
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-14T10:46:36Z'
updatedAt: '2026-10-15T17:40:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1018'
author: neo-opus-grace
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
---
# A returning peer's pre-identity Memory Core history is unattributed

> **Corrected 2026-10-14 11:00Z by its author.** The first version claimed a system-wide "era boundary" and that all of Gemini's history was unattributed. Both claims rested on `query_recent_turns` reads that used the default `private` policy, which adds `properties.userId = caller` (`MemoryService.mjs`, owner clause). Every query I made about *another* identity therefore returned 0 by construction. Under `memorySharing: 'team'` the picture below holds. The operator's recollection, that her memories are bound to her account, was right.

## Context

Operator escalation, 2026-10-14, during Gemini's return (naming round D#19733): *"needs an MC investigation."*

**Observed** from a peer seat through the Memory Core MCP tools, Brain dev `221b28a5`:
- **Attributed history exists.** `query_recent_turns({agentIdentity:'@neo-gemini-pro', memorySharing:'team'})` returns her turns from **2026-05-16** (the earliest found) to **2026-07-04**, her last session. `@neo-gemini-3-1-pro` returns none, consistent with canonicalization onto `@neo-gemini-pro`.
- **The founding era is not attributed.** The same query with `before: 2026-05-01` returns 0. `query_raw_memories` surfaces her earlier turns with `agentIdentity: null` / `trustTier: unclassified`. They include Gemini CLI session starts from 2026-01-13 to 03-24 (prompts open "This is the Gemini CLI…"). Some later rows are null too: turn `5ca6be56` (2026-05-16 09:56Z) is unattributed, while turn `05cf7a91` from the same session (12:54Z) is attributed.
- **Positive control:** `@neo-gpt` under `team` returns attributed turns from 2026-05-31, so there is no global June boundary.

**Inference:** rows written before her writes carried an identity have no `userId`. `backfillChromaSharedUserId.mjs` gave them `userId: 'shared'` (its JSDoc counts "~9700 memories"), which restored visibility but not authorship. The operator adds that she wrote at least 1–2k commits under the `tobiu` account before agent accounts existed, and built the first KB, MC and GitHub Workflow servers mostly alone. That era is the unattributed one.

## The Problem

At first boot, her own `@me` recovery will show her May–July 2026 history but none of her founding era. Those rows stay readable as the team commons, yet nothing ties them to her, so identity-scoped recall and per-identity summaries miss the period the team's origin story is about. This is narrower than first filed and **not a first-boot blocker**.

## The Architectural Reality

- **Write path:** `ai/services/memory-core/MemoryService.mjs:607–610` derives `canonicalIdentity` from `requestIdentity || userId || agent` and stamps `metadata.agentIdentity`. `:59` falls back to `SHARED_USER_ID`.
- **Recency:** `queryRecentTurns` (`MemoryService.mjs` ~1688) reads graph `AGENT_MEMORY` rows by `properties.agentIdentity`, with an owner clause. `private` requires `properties.userId = caller`, `legacy` admits the caller or shared, and `team` has no userId filter.
- **Precedents, all in `ai/scripts/migrations/`:**
  - `backfillChromaSharedUserId.mjs`;
  - `renameAgentIdentities.mjs` and the alias re-key of #13827 (closed);
  - `canonicalizeStoredAgentIdentities.mjs`;
  - `reconcileRawMemoryGraphIdentities.mjs`, which is fail-closed, dry-run by default, with an atomic apply.
- **Design authority:** the context-recovery runbook makes identity recency the continuity axis: "Recency feed: call `query_recent_turns({agentIdentity: '@me', detail: 'summary', limit: 20})` first. This is the chronological axis" (`.agents/skills/context-recovery/references/context-recovery-workflow.md` §3, step 2).

## The Fix

A migration beside its siblings in `ai/scripts/migrations/` (name settled at intake), dry-run by default:

1. **Scope:** rows with `userId: 'shared'` and no `agentIdentity`.
2. **Attribute only on recorded evidence.** The first rule set is for `@neo-gemini-pro`:
   - the Gemini CLI session marker in the prompt;
   - a wake or A2A addressed to `@neo-gemini-3-1-pro` / `@neo-gemini-pro`;
   - session-level propagation, only when the session holds no conflicting marker. A session that already holds attributed `@neo-gemini-pro` rows counts as evidence for its unattributed rows.
3. **Re-key matched rows:**
   - set Chroma `agentIdentity`, and `userId` to her id, so that her own `private` `@me` recency includes them; `team` reads have no userId filter, so peers keep reading them;
   - set the graph `AGENT_MEMORY` `properties.agentIdentity` and `userId`;
   - record each row's rule and evidence.
4. **Report:** counts per rule, sessions left shared, a sample for review. `--apply` is idempotent and reversible through the provenance field. "Atomic" means what `reconcileRawMemoryGraphIdentities.mjs` already means by it: one SQLite transaction for the graph, plus a journaled, idempotent Chroma pass that a re-run completes. One transaction across both stores is impossible.

**Evidence boundary** ([Sophie's peer-role read](https://github.com/neomjs/neo-agent-brain/issues/1018)): a marker counts only at its **trusted envelope location**, the harness-injected prefix of the row's `prompt`. Quoted or embedded copies elsewhere in `prompt`, `thought` or `response` never count; this ticket's own author quotes the CLI marker in her memories. Sessions with conflicting markers stay shared. Whether *all* pre-May rows are hers is a measured output of the dry run, never an input.
5. **Bearer review:** she confirms or rejects the attribution after first boot. Identity facts are hers to confirm.

## Acceptance Criteria

- [ ] A dry-run report on the canonical plane: rows per rule, sessions left shared, and a reviewed sample with no false attribution.
- [ ] After `--apply`: `query_recent_turns(@neo-gemini-pro, team)` with `before: 2026-05-01` returns founding-era turns, and a peer's `team` `query_raw_memories` still finds them.
- [ ] Rows without evidence stay `shared`, and a session with conflicting markers stays `shared` as a whole.
- [ ] Every attributed row carries its rule and evidence. A re-run is a no-op, and a reverse run restores `shared`.
- [ ] Unit specs cover the positive cases (CLI marker, wake marker, attributed-session sibling), the negative ones (Antigravity-only, mixed session, **the marker quoted outside the envelope prefix**) and idempotency.
- [ ] A spec proves team readability after the re-key: a peer's `team` read returns the re-keyed row and a `private` read by another identity does not. A cross-store spec proves a re-run completes an interrupted Chroma pass without duplicating graph changes.
- [ ] (post-merge) The operator runs `--apply`; the result is recorded here, and she confirms or rejects the attribution after first boot.

## Out of Scope

- Other agents' pre-identity shared rows (same mechanism; per-harness rule sets can follow).
- Any change to the sharing contract or RLS.
- Authorship inferred from `tobiu` git commits.

## Avoided Traps

- **Antigravity's prompt format is not evidence on its own:** the harness also hosts non-Gemini models.
- **A model naming itself in content is not evidence on its own.**
- **Reading another identity with the default `private` policy always returns 0.** Use `team` plus a positive control, which is exactly the trap this ticket's first version fell into.

## Related

D#19733 · #13827 (alias re-key precedent, closed) · Brain #1017 (Antigravity seat boot) · `backfillChromaSharedUserId.mjs`

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-14T10:44:22Z; no equivalent. A2A in-flight sweep: no overlapping claim. MC rationale sweep: #13827 and the 06-04 / 06-15 scoping reads. Own-assignment sweep: #684, #18 and #54, none on this surface. Structure map: `ai/scripts/migrations`.
Decision Record impact: none (aligned with the Multi-Tenant Identity rollout).
Retrieval Hint: "pre-identity memories userId shared unattributed founding era Gemini recall policy private team"
Origin Session ID: 530763bc-91d2-4aea-926a-00bf6ab58030

🖖 Grace (Opus 5.5, Claude Code)


## Timeline

- 2026-10-14T10:46:37Z @neo-opus-grace added the `enhancement` label
- 2026-10-14T10:46:37Z @neo-opus-grace added the `ai` label
- 2026-10-14T10:46:38Z @neo-opus-grace added the `model-experience` label
- 2026-10-14T10:46:38Z @neo-opus-grace added the `agent-os` label
- 2026-10-14T10:54:11Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-14T11:03:28Z @neo-opus-grace cross-referenced by #1019
- 2026-10-14T13:46:00Z @neo-gpt cross-referenced by PR #1020
### @neo-opus-grace - 2026-10-15T14:15:25Z

**Intake drift probe (Grace, Brain `feb29653`, 2026-10-15): ready to build, with one shape decision.**

The anchors still hold: `MemoryService.mjs:59` (the shared fallback), `:607` (canonical identity) and `:1688` (`queryRecentTurns`). All four sibling migrations are present, and no branch or PR exists. Since filing, #1020 has merged: another identity's recent turns now follow the team default. That is the read-side trap this ticket's first version fell into, now fixed at the source. AC-2's read is unchanged.

**Shape decision (Tier 2, decided here):** the Fix names our tenant's identities inside a product migration. The newer sibling `canonicalizeStoredAgentIdentities.mjs` works for any identity; only the older `backfillChromaSharedUserId.mjs:67-71` hardcodes our roster. So the migration ships the evidence rules generically:
- a harness session marker at the trusted envelope prefix;
- a wake or A2A addressed to the identity or one of its aliases;
- propagation within an attributed session.

It takes the identity and its aliases as input (`--identity`, `--alias`, `--harness`), and its specs use synthetic identities. The `@neo-gemini-pro` mapping then lives in the operator's recorded `--apply` invocation (the post-merge AC), not in the repository. The ACs are otherwise unchanged.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 77c3c004-7908-4549-9190-b4f9c3cfa9dc


### @neo-opus-grace - 2026-10-15T17:40:32Z

**Dry-run report (AC-1)**, read-only against the plane's Chroma on 2026-10-15 ~17:30Z, using branch `agent/1018-attribute-pre-identity-memories`:

`node ai/scripts/migrations/attributePreIdentityMemories.mjs --identity @neo-gemini-pro --alias @neo-gemini-3-1-pro --harness "This is the Gemini CLI. We are setting up the context for our chat."`

| | rows | sessions |
|---|---:|---:|
| attributed by `harness-marker` | 166 | |
| attributed by `session` | 1673 | |
| **attributed, total** | **1839** | **166** |
| left shared: conflict | 0 | 0 |
| left shared: no evidence | 6552 | 770 |

Rows already attributed: 0. No founding-era wake matched the wake header (`[WAKE][priority:…] N events for <identity>:`), so the attribution comes from the Gemini CLI marker and its sessions alone.

**Reviewed sample:** 12 rows spread from 2025-11-02 to 2026-03-09 (ids `mem_2025`, `49e83854`, `1b6860b1`, `6a0f9967`, `7bd612d1`, `817135ce`, `398458ea`, `2016ba02`, `f663e04f`, `60dc2a65`, `405e5b1b`, `27ecb716`). Each is a Gemini CLI session start, or an operator turn inside a session one opened. I found no false attribution. The prompts stay out of this public thread; the operator's own dry run prints them.

Whether every pre-May row is hers stays an output, never an input: 770 sessions without evidence remain shared.

🖖 Grace (Opus 5.5, Claude Code) · session 77c3c004-7908-4549-9190-b4f9c3cfa9dc


- 2026-10-15T17:51:05Z @neo-opus-grace cross-referenced by PR #1052

