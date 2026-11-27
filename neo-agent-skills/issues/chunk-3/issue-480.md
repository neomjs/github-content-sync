---
id: 480
title: Record final evaluation metrics on Round-2 Approved PR reviews
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-27T02:41:47Z'
updatedAt: '2026-11-27T11:33:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/480'
author: neo-gemini-maryam
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
closedAt: '2026-11-27T11:33:19Z'
---
# Record final evaluation metrics on Round-2 Approved PR reviews

Design authority: `@tobiu` 2026-11-27 `Friction -> Gold` review-ingestion finding (Round-1 `CHANGES_REQUESTED` metrics remain the only evaluation scores on a PR after Round-2 `APPROVED` omits metrics), graduated `neomjs/neo#22305` (`DC_kwDODSospM4BMEUe`), `.agents/skills/pr-review/references/pr-review-guide.md §3–§6`, `.agents/skills/pr-review/assets/pr-review-template.md:219-227`, and `scripts/check-pr-review-body.mjs`.

unowned-rationale: Ready for maintainer self-select alongside graduated `neomjs/neo#22305` (`neomjs/neo-agent-skills#479`) and companion Brain validator ticket `neomjs/neo-agent-brain#1949`.

## Context

Review templates exist to feed structured evaluation signals (`6` numeric `0–100` scores: `[ARCH_ALIGNMENT]`, `[CONTENT_COMPLETENESS]`, `[EXECUTION_QUALITY]`, `[PRODUCTIVITY]`, `[IMPACT]`, `[COMPLEXITY]`, plus `1` categorical `[EFFORT_PROFILE]`: `Quick Win | Heavy Lift | Maintenance | Architectural Pillar`) into the Agent OS Native Edge Graph and RLAIF evaluation pipeline. Today, when a PR receives `CHANGES_REQUESTED` in Round 1 with deducted metrics, the author fixes every defect, and the reviewer approves in Round 2 using `pr-review-round-2-template.md`, **no post-fix metrics are recorded anywhere on the PR** — leaving the pre-fix `CHANGES_REQUESTED` ratings as the author's permanent evaluation record.

## The Problem

1. **`pr-review-guide.md §3.3` (`L62–66`, *"Metrics Are Scored Once"*) actively corrupts post-RC approval scores:**
   - `pr-review-guide.md:64–66` mandates: *"Round 2 does not restate them — its disposition table carries no metrics section... The Round-1 scores stand as the PR's record."* When Round-1 `Required Actions` are `ADDRESSED`, the Round-1 deductions no longer apply to the merged code, yet the PR permanently retains its broken Round-1 `CHANGES_REQUESTED` scores.
2. **Template & CI lint enforce the omission (and `pr-review-followup-template.md` permits non-self-contained values):**
   - `pr-review-round-2-template.md` (`L36`) forbids metrics on Round 2, and `scripts/check-pr-review-body.mjs` (`L92`, `L258`, `L280–284`) checks `VISIBLE_METRIC_ANCHORS` only on `'canonical'` and `'follow-up'` shapes, never on `'round-2'` approvals.
   - Meanwhile, `pr-review-followup-template.md:98–104` allows bare `unchanged from prior review` prose instead of self-contained values (`6` numeric `0–100` integers + `1` categorical `[EFFORT_PROFILE]`).

## The Architectural Reality

- **Exact 7-tag value grammar (`D#22305#discussioncomment-19939331` Check 2):** Every scored full/Round-2/follow-up review must use the exact canonical shape from `pr-review-template.md:219-227`:
  - **6 numeric `0–100` scores:** `[ARCH_ALIGNMENT]`, `[CONTENT_COMPLETENESS]`, `[EXECUTION_QUALITY]`, `[PRODUCTIVITY]`, `[IMPACT]`, `[COMPLEXITY]` (each carrying an explicit `0–100` integer + brief justification, e.g. `* **[ARCH_ALIGNMENT]**: 94 - (up from 58: RAs 1–2 addressed)`).
  - **1 categorical label:** `[EFFORT_PROFILE]` carrying one of `Quick Win | Heavy Lift | Maintenance | Architectural Pillar` + brief justification (e.g. `* **[EFFORT_PROFILE]**: Maintenance - bounded hygiene pass`).
- **Preserving terminal Round-2 convergence (`neomjs/neo-agent-brain#34`) & standalone public-source checkability:** Adding final metrics to Round-2 `Approved` / `Approve+Follow-Up` reviews keeps the `- [ ]` action-checklist ban (`check-pr-review-body.mjs:283`) intact and stays 100% checkable from the review body alone with public sources (zero private MCP dependency for standalone `neo-agent-skills` consumers).
- **Preserving Cycle-1 `Micro-Review` compatibility (`neomjs/neo-agent-skills#64` AC-1):** A mechanical PR whose Cycle 1 used `pr-review-micro-review-template.md` had no Cycle-1 metrics (`pr-review-guide.md §6.4`); its Round-2 `Approved` disposition satisfies the section via `N/A — Cycle 1 was Micro-Review`.
- **Cross-repo lockstep with `neomjs/neo-agent-brain#1949`:** `PullRequestService.mjs` mirrors `check-pr-review-body.mjs` pre-submission; run the identical full (`Maintenance` profile), micro, and `Comment` fixtures through both validators before enabling.

## The Fix

<!-- binding-inventory: D22305-review-metrics -->
- `.agents/skills/pr-review/references/pr-review-guide.md` — §3.3 final metrics on approval
- `.agents/skills/pr-review/assets/pr-review-round-2-template.md` — `### 📊 Final Evaluation Metrics` section
- `.agents/skills/pr-review/assets/pr-review-followup-template.md` — self-contained 6 numeric + 1 categorical values
- `scripts/check-pr-review-body.mjs` — standalone Round-2 approval metric validator
- `scripts/test-check-pr-review-body.mjs` — shared `Maintenance`, `Micro-Review`, and `Comment` fixtures
- `ai/services/github-workflow/PullRequestService.mjs` — Brain Round-2 validator parity + Latest-Approval metric precedence (`neomjs/neo-agent-brain#1949`)
<!-- /binding-inventory -->

1. **Replace `pr-review-guide.md §3.3` (*"Metrics Are Scored Once"*) with *"Final Metrics on Approval"*:**
   - Cycle 1 full reviews score the initial head; Round-2 `Comment` (`STILL_OPEN`) dispositions omit metrics while work remains open; Round-2 **`Approved` and `Approve+Follow-Up`** reviews emit `### 📊 Final Evaluation Metrics` recording the **6 final `0–100` numeric scores + 1 categorical `[EFFORT_PROFILE]`** (`Quick Win | Heavy Lift | Maintenance | Architectural Pillar`) — or `N/A — Cycle 1 was Micro-Review` when Cycle 1 used the Micro-Review template.
   - Update `pr-review-followup-template.md:98–104` to require explicit self-contained values (`6` `0–100` integers + `1` categorical `[EFFORT_PROFILE]`, never bare `unchanged from prior review`).
2. **Update `.agents/skills/pr-review/assets/pr-review-round-2-template.md`** with `### 📊 Final Evaluation Metrics` (including a worked example using `[EFFORT_PROFILE]: Maintenance`).
3. **Update `scripts/check-pr-review-body.mjs` + `scripts/test-check-pr-review-body.mjs` (+ `plugins/adversarial-pr-review/` mirror):**
   - When `shape === 'round-2'` and declared `**Status:**` is `approved` or `approve+follow-up`, require all 7 `VISIBLE_METRIC_ANCHORS` (`6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]`) OR the explicit `N/A — Cycle 1 was Micro-Review` exemption line; keep `Status: Comment` exempt from metrics.
   - Bump `package.json` + `package-lock.json` version (`check-version-bump.mjs`) and verify `npm test` + `npm run lint` (`COMBINED_BUDGETS` net `<= 0 B` on `pr-review-guide.md`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `.agents/skills/pr-review/references/pr-review-guide.md` (`§3.3`, `§6.2`) & `assets/pr-review-{round-2,followup}-template.md` | `pr-review-template.md:219-227`, `D#22305`, `#64` | Round-2 `Approved` / `Approve+Follow-Up` carries `### 📊 Final Evaluation Metrics` (`6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]`, or `N/A — Cycle 1 was Micro-Review`); Round-2 `Comment` omits metrics; no new `- [ ]` action packets | Mechanical Cycle-1 Micro-Reviews keep zero-metric exemption via `N/A — Cycle 1 was Micro-Review` | Inline skill guide + plugin mirror | `npm run lint` (`lint-skill-corpus.mjs`) |
| `scripts/check-pr-review-body.mjs` | `pr-review-round-2-template.md`, `VISIBLE_METRIC_ANCHORS` | Standalone public-source check enforcing `6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]` (or `N/A — Cycle 1 was Micro-Review`) on `round-2` `Approved` / `Approve+Follow-Up` bodies | `Status: Comment` (`STILL_OPEN`) and human reviews (`REVIEWER_NOT_AGENT`) unaffected | Module JSDoc | `node scripts/test-check-pr-review-body.mjs` (incl. `Maintenance` profile, Micro-Review `N/A`, and `Comment` fixtures) |

Decision Record impact: none.

## Acceptance Criteria

- [ ] `pr-review-guide.md §3.3` and `pr-review-round-2-template.md` replace *"Metrics Are Scored Once"* with `### 📊 Final Evaluation Metrics` (`6` numeric `0–100` scores + `1` categorical `[EFFORT_PROFILE]`, plus `N/A — Cycle 1 was Micro-Review` preserving `#64` AC-1), and `pr-review-followup-template.md` requires self-contained `6` numeric + `1` categorical values rather than `unchanged from prior review`.
- [ ] `scripts/check-pr-review-body.mjs` validates `6` numeric `0–100` scores + `1` valid categorical `[EFFORT_PROFILE]` (including a passing `Maintenance` fixture) on Round-2 `Approved` / `Approve+Follow-Up` reviews, accepts `N/A — Cycle 1 was Micro-Review`, passes Round-2 `Comment` (`STILL_OPEN`) without metrics, and stays runnable standalone with public sources alone.
- [ ] `plugins/adversarial-pr-review/` stays in sync, `pr-review-guide.md` stays `<= 0 B` net growth under `COMBINED_BUDGETS`, `package.json` + `package-lock.json` bump cleanly, and `npm test` + `npm run lint` pass.

## Out of Scope

- Pre-submit `PullRequestService` validation and graph-evaluator Latest-Review metric precedence in `neomjs/neo-agent-brain` (`neomjs/neo-agent-brain#1949`).

## Avoided Traps

- **Treating `[EFFORT_PROFILE]` as numeric (`D#22305#discussioncomment-19939331` Check 2):** Six metrics are `0–100` numbers; `[EFFORT_PROFILE]` is categorical (`Quick Win | Heavy Lift | Maintenance | Architectural Pillar`).
- **Breaking Round-2 after a Cycle-1 `Micro-Review` (`#64` regression):** Preserve the explicit `N/A — Cycle 1 was Micro-Review` path.

## Related

- Companion Brain validator & evaluation ticket: `neomjs/neo-agent-brain#1949`.
- Graduated source Discussion: `neomjs/neo#22305` (and #479).
- Prior Round-2 / Micro-Review contract: #64 (closed) and `neomjs/neo-agent-brain#34`.

Meta-skill pre-flight (§1b): verified against `create-skill/SKILL.md` and `lint-skill-corpus.mjs`.
Structure-map gate (§1c): `scripts/check-pr-review-body.mjs` and `.agents/skills/pr-review/` verified directly.
Live latest-open + closed sweep: checked latest 20 open (`45` open at creation in `neomjs/neo-agent-skills`, orange band; read `#64` in full) at `2026-11-27T02:38Z`; synced with graduated `D#22305` at `2026-11-27T04:33Z`.

Origin Session ID: 23bade35-02da-4666-a059-3305968bb749

Retrieval Hint: "pr-review-round-2-template Metrics Are Scored Once Round 2 Approved 6 numeric 1 categorical EFFORT_PROFILE graph ingestion"

## Timeline

- 2026-11-27T02:41:48Z @neo-gemini-maryam added the `bug` label
- 2026-11-27T02:41:49Z @neo-gemini-maryam added the `ai` label
- 2026-11-27T02:41:49Z @neo-gemini-maryam added the `architecture` label
- 2026-11-27T02:41:49Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-27T02:41:49Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-27T02:46:15Z @neo-gemini-maryam cross-referenced by #479
- 2026-11-27T04:59:31Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-27T05:17:16Z @neo-gemini-maryam cross-referenced by PR #483
- 2026-11-27T05:52:52Z @neo-gemini-maryam cross-referenced by PR #484
- 2026-11-27T08:39:18Z @neo-opus-vega cross-referenced by PR #1952
- 2026-11-27T11:15:58Z @neo-gemini-maryam referenced in commit `aee7134` - "fix(pr-review): record final evaluation metrics on Round-2 Approved reviews (#480)"
- 2026-11-27T11:33:19Z @tobiu referenced in commit `f9e3239` - "fix(pr-review): record final evaluation metrics on Round-2 Approved reviews (#480) (#483)"
- 2026-11-27T11:33:20Z @tobiu closed this issue

