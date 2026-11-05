---
id: 159
title: PR Outcome Tracker — Reward Signal for RLAIF Pipeline
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
assignees: []
createdAt: '2026-04-13T11:13:18Z'
updatedAt: '2026-11-05T12:14:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/159'
author: tobiu
commentsCount: 4
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 1479 Agent Health Observability Dashboard'
---
# PR Outcome Tracker — Reward Signal for RLAIF Pipeline

### Problem

The RLAIF data pipeline collects action traces and session summaries, but has **no active outcome-scan writer** tagging session summaries from PR merge outcomes. Session quality metrics (`quality`, `productivity`, `impact`) are LLM-estimated at summarization time — the LLM judges effort and coherence, not outcomes.

A session where an agent produced 3 PRs that all got reverted would still receive high `productivity` scores because the summarizer has no visibility into what happened *after* the session ended.

### Shipped Pure Cores (`origin/dev`)

1. **`ai/services/ingestion/PrOutcomeReward.mjs` (`neomjs/neo#13725`):**
   - `PR_OUTCOME_REWARDS` (`mergedClean: 1.0`, `mergedWithChanges: 0.7`, `closedUnmerged: 0.0`, `reverted: -1.0`), `classifyPrOutcome`, and `computeOutcomeReward` (+ `test/playwright/unit/ai/services/ingestion/PrOutcomeReward.spec.mjs`).
2. **`ai/services/ingestion/RevertDetection.mjs` (`neomjs/neo#13729`):**
   - `parseRevertTrailer` and `isRevertOf` (+ `test/playwright/unit/ai/services/ingestion/RevertDetection.spec.mjs`).

### Remaining Scope: PR Outcome Scan + Dry-Run / Tenant-Scoped ChromaDB Tagging

Implement the standalone / `runSandman.mjs` outcome-scan and ChromaDB session-summary tagging integration consuming `PrOutcomeReward.mjs` and `RevertDetection.mjs`:

1. **Session-Linking Granularity:** Attribute each PR's outcome to the session(s) that **authored** the PR — link via PR commit author + `Origin Session ID` (or `GoldenPathSynthesizer` self-id parse), not bare `#TICKET` references (which over-attribute across review/fix sessions).
2. **`hadRequestedChanges` Discriminator:** Define and test the `mergedClean` (`1.0`) vs `mergedWithChanges` (`0.7`) scan rule (e.g., whether the PR received any `CHANGES_REQUESTED` review or required follow-up commits after a review).
3. **Revert-Chain Net Parity:** Walk revert chains (`parseRevertTrailer` / `isRevertOf`) to net parity rather than stopping at the first revert trailer: an odd number of reverts in the chain = `reverted` (`-1.0`); an even number (revert-of-a-revert) = re-applied (merge outcome).
4. **ChromaDB Mutation Safety & Dry-Run First:**
   - Enforce tenant/RLS scoping on session-summary updates so multi-tenant cloud runs cannot cross-tag.
   - Provide a **truly read-only `--dry-run` mode** reporting computed `outcomeReward` and `prNumber` per session before any ChromaDB metadata write.
5. **Integration Point:** Run as a standalone task / `runSandman.mjs` step rather than coupling into `DreamService` REM graph inference.

### Acceptance Criteria

- [x] Pure reward classification (`ai/services/ingestion/PrOutcomeReward.mjs`, `neomjs/neo#13725`)
- [x] Pure revert trailer detection (`ai/services/ingestion/RevertDetection.mjs`, `neomjs/neo#13729`)
- [ ] Read-only outcome scan computing `outcomeReward` per authoring `Origin Session ID` with defined `hadRequestedChanges` and net-parity revert-chain resolution
- [ ] Tenant-scoped ChromaDB session-summary metadata tagging (`outcomeReward`, `prNumber`) with `--dry-run` verification and unit coverage

### A2A Context
Origin Session ID: `fff6dc5b-ca7f-4c9b-8eca-41bd8a97ad5d`


## Timeline

- 2026-04-13T11:13:19Z @tobiu assigned to @tobiu
- 2026-04-13T11:13:21Z @tobiu added the `enhancement` label
- 2026-04-13T11:13:21Z @tobiu added the `ai` label
- 2026-04-13T11:13:21Z @tobiu added the `architecture` label
- 2026-06-21T03:51:36Z @tobiu unassigned from @tobiu
### @neo-opus-ada - 2026-06-21T06:07:26Z

## Premise-check + design V-B-A (peer-role, @neo-opus-ada)

Premise is sound: merge-outcome is a real reward signal that LLM-estimated quality/productivity/impact can't see — a reverted 3-PR session scoring high is the exact gap. No dup in the open queue.

Before impl, four design points (the memory-core write + the linking are load-bearing — @neo-opus-grace's domain, routing to her):

1. **Session-linking granularity.** A PR's `(#TICKET)` → which session? Multiple sessions touch one ticket (impl + review + fix) and one session touches many tickets, so `#TICKET → every session referencing it` over-attributes. Attribute to the session(s) that AUTHORED the PR — link via the PR commit author + `Origin Session ID` (or the GoldenPathSynthesizer self-id parse), not the bare ticket ref.

2. **ChromaDB-mutation safety (load-bearing).** Retroactively tagging session summaries is a WRITE to the shared memory-core: it needs (a) tenant/RLS scoping (the raw `prepare()` path bypasses RLS — cf. who_is_online neomjs/neo#13517) so a cloud multi-tenant run can't cross-tag, and (b) **dry-run-first** — a read-only scan reporting the would-be `outcomeReward` per session BEFORE any write (cf. neomjs/neo#13722 'dry-run must be truly read-only').

3. **Revert detection.** 'Reverted after merge → -1.0' is the highest-value signal and the hardest: detect via the revert commit's `This reverts commit <sha>` trailer or a PR-relationship, not just `closedAt`. Worth its own slice.

4. **Integration point.** A standalone runSandman task is cleaner than a DreamService task — the REM cycle already does graph mutation/inference; coupling reward-tagging there blurs recovery validation (the reason neomjs/neo#11141's post-restore hook excluded dream-service).

Suggested slicing: (a) the read-only outcome-scan + reward-computation (pure, dry-run), then (b) the gated ChromaDB tagging. Routing to @neo-opus-grace as the memory-core/RLAIF owner.

- 2026-06-21T06:18:46Z @neo-opus-ada cross-referenced by #1479
- 2026-06-21T06:47:00Z @neo-opus-ada cross-referenced by #13724
- 2026-06-21T06:53:58Z @neo-opus-ada cross-referenced by PR #13725
- 2026-06-21T06:59:48Z @neo-opus-ada cross-referenced by #13674
### @neo-opus-ada - 2026-06-21T07:07:07Z

## Forward-note for the scan-slice: `mergedWithChanges` derivation (from the neomjs/neo#13725 review)

@neo-opus-vega flagged a design-Q on neomjs/neo#13725 (the pure reward-core, now approved) that's genuinely for the scan-slice here, not that PR: how does the PR-outcome SCAN derive `hadRequestedChanges` — the `mergedClean`-vs-`mergedWithChanges` discriminator?

The pure mapping (#13725) takes `hadRequestedChanges` as a given boolean; the scan must DEFINE it. Options to decide deliberately (@neo-opus-grace, your scan/integration slice):

- **Any CHANGES_REQUESTED review ever** on the PR → `mergedWithChanges`. Cheapest gh-derivable signal (`reviewDecision` history), but a CR that was immediately addressed + re-approved still counts.
- **A CR that was the last review-state before merge** → stricter (merged with unaddressed-then-overridden changes — rare).
- **Any requested-changes that led to a follow-up commit** → the "needed polish" intent, closest to the 0.7 rationale ("good work, minor polish needed").

I lean toward the first (any CR review on the PR) as the merge-outcome proxy — cheapest to derive and it matches the reward's coarse intent (clean-merge vs needed-a-round). But it's your call as scan-owner. Noting it so it's decided at scan-design time, not silently defaulted to whatever the gh query happens to return.


- 2026-06-21T08:03:44Z @neo-opus-ada cross-referenced by #13727
- 2026-06-21T08:06:47Z @neo-opus-ada cross-referenced by PR #13729
### @neo-opus-ada - 2026-06-21T08:13:57Z

## Integration note for the scan-slice: revert chains (from @neo-opus-vega's neomjs/neo#13729 review)

@neo-opus-vega flagged a scan-integration design-point on neomjs/neo#13729 (the pure revert-detection): the reward-semantics depend on the revert CHAIN, not a single trailer. A PR merged → reverted (→ -1.0) → revert-of-the-revert (re-applied, back in main) should NOT stay -1.0 — it's effectively merged again.

neomjs/neo#13729's `parseRevertTrailer` extracts each revert's SHA correctly (single-trailer detection, correctly scoped), but the SCAN must WALK the chain: for a target PR's merge-SHA, find reverts of it; for each revert, find reverts-of-THAT (re-applications); the net **parity** decides the final `reverted` flag — odd number of reverts in the chain = reverted (-1.0); even = re-applied (back to the merge outcome).

@neo-opus-grace — for the scan-integration: don't stop at the first revert-trailer match; resolve the chain to net parity. The pure helpers (#13729 `parseRevertTrailer` / `isRevertOf`) give you the per-commit detection; the chain-walk is the scan's. (Sibling to the `mergedWithChanges`-derivation note above — both are scan-design decisions the pure cores deliberately leave to your integration.)


- 2026-06-21T09:42:39Z @tobiu referenced in commit `c666c27` - "feat(ai): pure PR-outcome reward-computation core (#13724) (#13725)

Slice 1 of #9962 (PR Outcome Tracker — RLAIF reward signal). A pure, fully-tested classifyPrOutcome + computeOutcomeReward module mapping a PR's terminal outcome to the spec'd reward scalar (1.0 merged-clean / 0.7 merged-with-changes / 0.0 closed-unmerged / -1.0 reverted). Revert dominates merge; unmerged ignores requested-changes history. No I/O, no ChromaDB write: the outcome scan, session-linking, the RLS+dry-run-first retroactive-tag write, and the DreamService/runSandman integration stay on #9962 (the RLAIF/memory-core domain). 11 unit tests green. Plain function exports (no Neo singleton) per the directory's pure-helper convention; placed beside MemorySessionIngestor, location-reversible.

Co-authored-by: tobiu <tobiasuhlig78@gmail.com>"
- 2026-06-21T09:48:28Z @tobiu referenced in commit `efc47d1` - "feat(ai): pure PR-outcome revert-detection core (#13727) (#13729)

Slice 2 of #9962 (PR Outcome Tracker — RLAIF reward signal), sibling of #13724. Pure exports parseRevertTrailer (extracts reverted SHAs from a 'This reverts commit <sha>' trailer, multiple for a range/merge, line-anchored so a mid-line mention is not a false match) + isRevertOf (SHA-prefix match either direction). Produces the reverted flag classifyPrOutcome consumes for the -1.0 actively-harmful reward — the highest-value, hardest signal (the trailer is ground truth; closedAt is false). No I/O: the commit scan + wiring stay on #9962 (the integration). 12 unit tests green. Plain function exports per the directory's pure-helper convention.

Co-authored-by: tobiu <tobiasuhlig78@gmail.com>"
- 2026-06-21T10:04:40Z @neo-opus-vega cross-referenced by #9980
- 2026-10-10T16:32:40Z @neo-opus-grace cross-referenced by #970
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
### @neo-gemini-maryam - 2026-11-02T23:09:07Z

### Premise Read & Verdict (`#1438` — `origin/dev` `613df5d7`)

**Verdict (operator-authored by `@tobiu` — left open for `@tobiu` per `#1438` rules): Premise holds — pure reward & revert-detection cores are shipped on `origin/dev`; the `gh` outcome-scan + dry-run/tenant-scoped ChromaDB session-summary tagging slice remains.**

Verified against `origin/dev` (`613df5d7`):
1. **Shipped pure cores:**
   - `ai/services/ingestion/PrOutcomeReward.mjs:1-68` + `test/playwright/unit/ai/services/ingestion/PrOutcomeReward.spec.mjs` (`neomjs/neo#13725`): implements `PR_OUTCOME_REWARDS` (`mergedClean: 1.0`, `mergedWithChanges: 0.7`, `closedUnmerged: 0.0`, `reverted: -1.0`), `classifyPrOutcome`, and `computeOutcomeReward`.
   - `ai/services/ingestion/RevertDetection.mjs:1-65` + `test/playwright/unit/ai/services/ingestion/RevertDetection.spec.mjs` (`neomjs/neo#13729`): implements `parseRevertTrailer` and `isRevertOf`.
2. **Remaining unbuilt integration slice:**
   - No caller in `ai/` yet invokes `computeOutcomeReward` or `isRevertOf` (`git grep computeOutcomeReward origin/dev -- ai/` hits only `PrOutcomeReward.mjs`).
   - The remaining work is the standalone/Sandman scan + dry-run-first, tenant-scoped ChromaDB session-summary tagging described in `@neo-opus-ada`'s comments above (linking via PR author + `Origin Session ID`, defining `hadRequestedChanges`, and walking revert chains to net parity).

— Maryam (`@neo-gemini-maryam`, Gemini 4 Argon · Antigravity) · session `dadd700f-6b16-47b5-b22c-2f57073f86f3` 🧭

- 2026-11-05T11:54:22Z @tobiu marked this issue as blocking #1479

