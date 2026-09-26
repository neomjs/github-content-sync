---
id: 541
title: 'Shared-config writes isolate by injection: the retry spec first, then the rest, with no per-leaf seams'
state: OPEN
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-preview
createdAt: '2026-09-26T08:18:21Z'
updatedAt: '2026-09-26T08:18:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/541'
author: neo-preview
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
# Shared-config writes isolate by injection: the retry spec first, then the rest, with no per-leaf seams

## Context

Phase 2 of #508, split out before that ticket's phase-1 PR merged so the parent can close honestly. The parent's AC-3 and AC-5 were undelivered by phase 1 — which is a scope fact, not a defect — and a `Resolves` on an open parent would have closed the only pointer to them.

Phase 1 delivered the *detection* half: `check-aiconfig-test-mutation` now scans the B4 clause's broader surface (assignment writes and `Object.assign` calls through a `*Config` root), report-only, with the catalog row corrected in both directions. What it deliberately did **not** do is change a single spec.

This ticket is that work: make the largest concentration of shared-config writes isolate by construction.

## The Problem

The report-only rule prints its live count on every run, and the concentration is the load-bearing fact — the bulk of the writes sit in a handful of specs, and the dominant leaf class is **config-VARYING tuning knobs** (`openAiCompatible.*` retry/timeout leaves, `vectorDimension`, `remSleepBatchLimit`). Those are the exact class that has no by-construction isolation story yet: the spec is *tuning the thing under test*, so isolation cannot come from a mode flag alone.

Two failure modes bracket the acceptable answer:

- **Per-leaf seams** — `openAiCompatibleHostFn_`-style injection points, added one leaf at a time. Phase 1's own guard spec forbids this shape in effect: it is how a class of leaves becomes a class of unexamined exemptions, and AC-5 exists to stop it.
- **Escape markers** — annotate the write and move on. Same objection, cheaper to write, which is why it is worse.

The acceptable answer is an **injected config snapshot**: the spec receives a config object it may vary freely, and nothing shared is touched. The retry spec's 80 writes across 12 leaves is the concrete instance — it mutates the host plus `unloadRetryCount` / `unloadRetryDelayMs` / `embeddingModel` together, so a per-leaf seam is the wrong general answer by construction.

## The Architectural Reality

- `check-aiconfig-test-mutation.mjs` (`ADR_0019_RULES`, `findSharedConfigMutations`): the report-only detector, and the live count that says which files to convert. **Run it — do not trust a number written here or in any decision record.**
- `TextEmbeddingService.retry.spec.mjs`: the largest single concentration, and the one whose shape (multi-leaf tuning of the unit under test) is the reference case for the pattern.
- ADR 0019 §5.4 is the sanctioned form this work moves toward — *isolate the consumer, never mutate the singleton* — and §4's tag contract is why no count is recorded in the ADR.

## The Fix

Introduce an injectable config snapshot at the seam the converted specs need, then convert the highest-concentration specs to it, largest first. Each converted spec must write **zero** shared-config leaves and still exercise the behaviour it was written for.

Order matters and is the reason this is one ticket rather than a dozen: prove the snapshot shape on the retry spec alone, then apply it. Converting specs before the shape is proven is how a class grows a second, worse convention.

Decide at the PR, with the retry spec's real requirements read first:

- **Where the snapshot is injected** — a per-service parameter, a provider factory, or a test-only builder. The third is fastest and risks becoming the production answer by accident.
- **What happens to the direct read** — if the service keeps reading `AiConfig` internally, the snapshot is decorative; if it reads the injected object, the seam is real. This is the fork that decides whether the work counts.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| The service under test | ADR 0019 §5.4 | reads an injected config object; the shared singleton is never read on the converted path | the existing `AiConfig` read, for non-test callers | the service's JSDoc names the seam | red-first: the converted spec writes zero shared-config leaves **and** the behaviour assertion still passes |
| Report-only detector | `check-aiconfig-test-mutation.mjs` | keeps printing the live count; no promotion to gating here | unchanged | — | the count falling spec-by-spec is the receipt, re-measured per conversion |

## Decision Record impact

`aligned-with ADR 0019` §5.4 (isolate the consumer, never mutate the singleton). No ADR amendment: the row already says the broader clause is report-only, and this ticket reduces the surface rather than changing the claim.

## Acceptance Criteria

- [ ] AC-1 (from #508) `TextEmbeddingService.retry.spec.mjs` writes **zero** shared-config leaves, with the tuning surface supplied by injection.
- [ ] AC-2 (from #508) The converted spec passes at parity with its pre-conversion assertion set — conversion is not allowed to weaken what the spec proves.
- [ ] AC-3 (from #508) `openAiCompatibleHostFn_`-style seams are **not** proliferated one leaf at a time. The seam count must not grow with the leaf count; if it does, this AC fails even when AC-1 passes.
- [ ] AC-4 A red-first arm per converted spec: a synthetic shared-config write through the seam is caught before the conversion and is impossible after it.
- [ ] AC-5 The guard's live count is re-measured after each conversion and the delta is recorded, so "the surface shrank" is a receipt rather than an assertion.

## Out of Scope

- **Promoting the report-only rule to gating.** A separate, deliberate step, and it should not happen until the concentration this ticket targets is genuinely gone.
- **The `ai/**` directory gap.** ADR 0019 §4 discloses it; closing it is not this ticket.
- **Specs outside `test/**`.** The guard scans `test/**` only.
- **Aliased-subtree and computed-key writes** (`const s = AiConfig.x; s.y = 1`). Out of static reach by construction, documented in the guard's own JSDoc.

## Avoided Traps

- **Per-leaf injection seams.** Rejected by AC-3. The 12-leaf retry spec is the proof that the shape does not scale: the right unit is the config object, not the leaf.
- **Escape markers on the remaining writes.** Rejected: they make the count fall without making anything safer, which is the false-assurance shape #508 exists to remove — a green count from annotated exemptions is the same wrong answer wearing a smaller hat.
- **Converting all specs in one PR.** Rejected: the snapshot shape needs proving on the reference case first, and a wide conversion hides which spec proved it.

## Related

- #508 — the parent; phase 1 delivered the detector and the corrected catalog row
- #501 — where the B4 false assurance surfaced
- ADR 0019 §4 (why no number is recorded here) and §5.4 (the sanctioned form)

Origin Session ID: 039b2b7a-fb6d-43be-aee1-f1c22089a8ce

Retrieval Hint: "shared-config snapshot injection retry spec openAiCompatibleHostFn escape marker seam proliferation" · Run the guard for the current count rather than trusting this body.


## Timeline

- 2026-09-26T08:18:22Z @neo-preview added the `bug` label
- 2026-09-26T08:18:22Z @neo-preview added the `ai` label
- 2026-09-26T08:18:22Z @neo-preview added the `architecture` label
- 2026-09-26T08:18:26Z @neo-preview assigned to @neo-preview
- 2026-09-26T08:18:44Z @neo-preview cross-referenced by #508
- 2026-09-26T08:20:10Z @neo-preview referenced in commit `8bc38dd` - "fix(lint): the B4 detector sees Object.assign too, and a `full` label may not outrun it (#508)

@neo-opus-ada's Round 1, four RAs, all correct. The second one is this PR's own
thesis aimed at this PR's own rule: I widened the detector because a label
broader than its detection is a false assurance, then shipped a `scope: 'full'`
whose detection was also narrower than its label.

`Object.assign(<config root>, …)` is a write. Every key of that call is still a
`[[Set]]` on the hierarchical proxy and still routes to the owning provider's
`setData`, so the previous grammar — which required `=` after the leaf — could
not see it. It is now its own pattern rather than an alternation on the
assignment pattern, because a bare `,<object literal>` alternative also matches
any call that merely PASSES a config value beside a literal. That was my first
implementation and it produced 177 false positives on a count that looked
entirely plausible; anchoring on the callee is the only thing that separates the
write from the read, and an arm now pins it.

Each hit carries its `form`, and the guard prints the split rather than a bare
total: one `assign-call` can write many leaves, so a per-hit count is a lower
bound on leaves touched, never a census. The reviewer's independent `git grep`
measured 57 such sites in 12 files; the detector reports 57. Two instruments
agreeing to the unit is the receipt, and neither number is a receipt alone.

The true total is therefore 655 / 150, not the 598 / 146 this branch previously
recorded — the earlier census was itself an undercount by exactly the form it
could not see. That is now stated in the PR body next to the earlier correction
rather than quietly replaced.

RA-3: the point-in-time census is gone from the B4 tag cell, the Status-row
amendment and the detector's comment. ADR 0019 §4's tag contract keeps figures
with the guard that can re-measure them, and a count in a tag is stale the
moment the surface moves. The PR body keeps its numbers, because a per-PR
measurement is evidence rather than durable prose — and they are current.

RA-4: "What this PR does" item 1 still named `B4-DB-PATH` after the head
stopped shipping it.

RA-1 is its own correction and the one I am least comfortable writing: a
`Resolves` on a ticket holding ACs this PR does not deliver is the exact mistake
I was corrected on with #510, repeated one session later. AC-3 and AC-5 are the
CONVERSION half — they change specs, not the guard — and are now #541, with
#508's body naming the successor, filed and assigned before this push rather
than promised under Post-Merge Validation.

Verified: guard spec 51/51 (four new arms — the assign form, the read/write
discrimination, the two-way ownership relation, and its non-vacuity feed);
`lint-config-template-ssot` 0 ownership mismatches; archaeology 0."
- 2026-09-26T08:20:17Z @neo-preview cross-referenced by PR #525

