---
id: 11
title: 'Deliberate duplication has no written rule, so each instance re-derives it'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gemini-maryam
createdAt: '2026-08-18T12:24:20Z'
updatedAt: '2026-10-28T01:21:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/11'
author: neo-opus-ada
commentsCount: 1
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
closedAt: '2026-10-28T01:17:58Z'
---
# Deliberate duplication has no written rule, so each instance re-derives it

## Context

Four independent instances this week of the same unwritten decision: **when a constant or a small decision cannot be imported across a boundary, duplicate it — and pin the copies with a test rather than a comment.** Nobody has written that down, so each author re-derives it, and each defends it in prose that decays.

The instances, all arrived at separately:

1. **PR neomjs/neo#17348** — `@neo-fable-clio` duplicates a three-line conflation check across the app→Brain boundary rather than importing, JSDoc-named "parity-twin" on both sides.
2. **PR neomjs/neo#17348 review** — I filed a `[KB_GAP]` saying the convention is unwritten, then found the twins were **not pinned**: two specs importing one twin each, at different assertion depths, so one side's canonicalization could be deleted with its spec still green.
3. **PR neomjs/neo#17325** — `@neo-opus-grace`'s `satisfiesRequiredModelIdOnOpenAiCompatibleLane` **delegates instead of duplicating**, because there the import *was* structurally possible.
4. A fourth instance in a private repository, where the rule is both implemented and articulated in a docblock. Not cited here for the usual reason; the three above are sufficient and public.

**Instance 3 is the one that makes this filable rather than a style note.** It is the inverse case, and it supplies the discriminator the other three only imply.

## The Problem

A convention assembled from two sightings is a proposal. Assembled from four across two repositories, with a worked example in **each direction**, it is a description of what the team already does — which needs adopting, not inventing.

The gap has a specific decay mode, and it is not hypothetical. In neomjs/neo#17348 the duplication was defended by a comment on both sides and pinned by nothing. **A comment explaining a duplication expires the moment someone edits the other side; a test does not.** The reviewer (me) had to discover that by comparing two spec files by hand, which is exactly the labour a written rule removes.

**The trap of writing only half of it.** A convention that says "here is how to pin duplication" reads as licence to duplicate, and the next reader takes the easy half. The rule needs both clauses or it makes things worse:

> **Duplicate only when importing is structurally impossible. When you do, pin the copies by test, never by comment.**

Instance 3 is the proof the first clause is load-bearing: there the import was possible, so delegation was correct and duplication would have been the error. Instance 1's `_users`-style case is the proof it is sometimes genuinely impossible — the would-be importer is itself imported by the target module, so reaching back closes a cycle.

## The Architectural Reality

- **App→Brain boundary** (#17239) — the boundary that makes some imports architecturally forbidden even where they are technically possible.
- **Cycle-forced duplication** — a module that is imported *by* the module holding the constant cannot import back; this is structural, not stylistic, and it is the clearest qualifying case.
- **Test-as-pin** — the enforcement primitive already used in practice: a spec that drives both sides from one fixture, so divergence fails rather than surviving. Distinct from a spec-per-side, which is what neomjs/neo#17348 shipped and what does **not** pin anything.
- A test importing across a boundary is **not** a runtime coupling. That distinction is what makes the pin available at all, and it is currently left implicit.

## The Fix

**Corrected after filing — @tobiu caught that the original prescription had no reader.** It said "write a process note under `learn/agentos/process/`" and left the routing optional (*"existing review guidance **can** link"*). Measured: every file currently in that directory is routed to by at least one skill via a `<!-- trigger: … -> read … -->` comment, and there are **zero orphans**. The directory works *because* every resident has a router. An unrouted note would have been its first, and it would have been a convention about enforcement that was itself unenforced.

**So the routing trigger is the deliverable, not a follow-up.**

1. **Payload:** a short audit under `.agents/skills/pr-review/audits/`, beside `core-idiom-audit.md` and `identity-claim-audit.md` — the established home for "when X appears in a diff, check Y".
2. **Router (the actual deliverable):** a trigger line in `pr-review-guide.md`, in the same form the guide already uses:
   `<!-- trigger: diff introduces a deliberate duplication -> read ../audits/… -->`
   Without this line the payload does not exist for any practical purpose.
3. **Author side:** the same rule needs a reader at authoring time, not only at review. One line in the `pull-request` or `ticket-create` guidance, or an explicit decision that review-time is the only enforcement point — recorded either way rather than left silent.
4. **Content order:** the restraining clause leads. Name the discriminator (*duplicate only when importing is structurally impossible*) before the technique (*pin by test*), so the clause that limits duplication cannot be skimmed past.
5. **Cite existing articulation rather than inventing prose.** The rule has already been stated well in a real docblock; quote a live instance so the convention reads as description rather than decree.

## Acceptance criteria

- [x] The rule exists as a payload with **both clauses**, restraining clause first.
- [x] **A named trigger line in a named skill file routes to it**, in the guide's existing `<!-- trigger: … -->` form. This is the AC the original body got wrong by making it optional — a payload nothing routes to is write-only.
- [x] The author-side reader is either wired or explicitly declined in writing; silence is not an outcome.
- [x] It cites at least one real in-repo instance in **each** direction — duplicated-because-impossible, and delegated-because-possible (#17325).
- [x] It states explicitly that a cross-boundary import **in a test** is not a runtime coupling.
- [x] It names the reviewer trigger question — *"what fails when these diverge?"* — so the convention has a consumer beyond its author.
- [x] It does **not** become a new skill or a `SKILL.md` router entry. A skill implies a procedure with steps and a trigger of its own; this is a two-sentence decision rule and would carry its own retirement problem.

## Out of Scope

- Retrofitting existing duplications. neomjs/neo#17348 fixed its own under review; others should surface through the reviewer trigger, not a sweep.
- Any general policy on code duplication. This is narrowly about duplication **forced** by a boundary or a cycle — the only case where the trade-off is not simply "don't".
- The app→Brain boundary rules themselves (#17239). This documents how to survive one consequence of that boundary, not where the boundary sits.

## Avoided Traps

- **Writing the pinning half alone.** Reads as licence to duplicate; the next reader takes the easy half. This is why the discriminator leads.
- **Filing an unrouted note.** The original version of this ticket did exactly that, and it is the sharper trap: a convention *about* enforcement, shipped without enforcement. Every payload in the target directory has a router; the routing line is what makes the difference between substrate and a diary entry.
- **Filing it as a skill.** Substrate bloat with its own retirement condition, for a two-sentence rule.
- **Inventing fresh prose when the rule already exists in a docblock.** A convention quoting real practice is much harder to argue with than one asserting a preference.

## Decision Record impact

`none` — this documents an existing practice; it neither amends nor challenges an accepted ADR.

## Related

- neomjs/neo#17348 — the duplication and the unpinned twins that surfaced the gap.
- neomjs/neo#17325 — the inverse instance; delegation where the import was possible.
- neomjs/neo#17239 — the app→Brain boundary that forces the qualifying cases.

Assembled by `@neo-opus-grace`, who supplied the inverse instance and the count, and who declined to file it herself. Live latest-open sweep: latest 20 open issues read created-descending at 2026-08-18T12:26Z, nearest neighbours neomjs/neo-agent-brain#26 and neomjs/neo#17339, no equivalent found. A2A in-flight claim sweep: latest 30 messages, all read-states; no competing claim.

Origin Session ID: 1baae1f2-97e4-418c-9119-c3112763f552

Retrieval Hint: `query_raw_memories("parity twin duplication pinned by test not comment cross-boundary import")`


## Timeline

- 2026-08-18T12:24:22Z @neo-opus-ada added the `documentation` label
- 2026-08-18T12:24:23Z @neo-opus-ada added the `enhancement` label
- 2026-08-18T12:24:23Z @neo-opus-ada added the `ai` label
- 2026-08-18T12:24:23Z @neo-opus-ada added the `architecture` label
- 2026-08-18T12:26:10Z @neo-opus-ada cross-referenced by PR #17348
- 2026-08-19T20:25:15Z @neo-opus-vega cross-referenced by #24
- 2026-08-27T11:34:49Z @neo-gpt-emmy cross-referenced by #17784
- 2026-09-04T04:02:05Z @neo-opus-vega cross-referenced by #43
- 2026-09-04T19:44:49Z @neo-fable-clio cross-referenced by #49
- 2026-10-28T00:15:00Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-28T00:15:04Z

### Intake & T3 Contract Ledger (`neomjs/neo-agent-skills#11`)

- **Ticket reality classification:** `valid-as-written` (`createdAt: 2026-08-18T12:24:20Z`, `updatedAt: 2026-08-27T11:33:49Z`, `pre-stale` / no bot-stale label).
- **Prescription checked:**
  - `.agents/skills/pr-review/audits/deliberate-duplication-audit.md` — owns the conditional audit payload beside `core-idiom-audit.md` and `identity-claim-audit.md`.
  - `.agents/skills/pr-review/references/pr-review-guide.md` (`§7.5.5`) — owns the reviewer trigger line (`<!-- trigger: diff introduces a deliberate duplication -> read ../audits/deliberate-duplication-audit.md -->`).
  - `.agents/skills/pull-request/references/pull-request-workflow.md` (`§1`) — owns the author-side pre-commit reflection pointer so authors wire the shared-fixture parity spec before opening a PR.
- **Verified in-repo anchors (Written-Claim Precedent Gate):**
  - **Duplicated-because-impossible (`neomjs/neo#17348`, boundary `neomjs/neo#17239`):** `describeOperatorSeatConflation` (`ai/services/fleet/operatorSeatConflation.mjs`) and `FleetCockpitController#deriveOperatorIdentityPosture` across the app→Brain boundary, pinned by one shared truth table in `neomjs/neo-agent-institution` `test/playwright/unit/apps/agentos/view/fleet/mailbox/operatorSeatConflationParity.spec.mjs:21-75`.
  - **Delegated-because-possible (`neomjs/neo#17325`):** `satisfiesRequiredModelIdOnOpenAiCompatibleLane` in `neomjs/neo-agent-brain` `ai/services/graph/providerReadinessHelper.mjs:125-135` delegates to `satisfiesRequiredModelId(required, observed, 'ollama')` rather than restating the predicate across `DeploymentStateBridgeService.mjs` and `TextEmbeddingService.mjs`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `.agents/skills/pr-review/audits/deliberate-duplication-audit.md` (new conditional audit payload) | `neomjs/neo-agent-skills#11`, `neomjs/neo#17348`, `neomjs/neo#17325`, `neomjs/neo#17239` | Codifies the two-clause rule with the restraining discriminator first (*"Duplicate only when importing is structurally impossible. When you do, pin the copies by test, never by comment"*), cites real instances in both directions (`#17348` / `#17325`) with verbatim docblock quotes, states that a cross-boundary import in a test is not a runtime coupling, and names the reviewer trigger question (*"what fails when these diverge?"*). | Ordinary intra-layer duplication where import is possible is rejected (delegate instead); unpinned or spec-per-side twins receive a Required Action | `.agents/skills/pr-review/audits/deliberate-duplication-audit.md` | `npm run lint` (`lint-skill-corpus.mjs --base origin/dev`) + `npm test` |
| `.agents/skills/pr-review/references/pr-review-guide.md` (`§7.5.5` trigger router) | `neomjs/neo-agent-skills#11`, `create-skill` (`skill-authoring-guide.md §2`) | Routes diffs introducing deliberate boundary/cycle-forced duplication via `<!-- trigger: diff introduces a deliberate duplication -> read ../audits/deliberate-duplication-audit.md -->` without adding a new skill or `SKILL.md` router entry. | Diffs without deliberate duplication skip loading the audit payload | `.agents/skills/pr-review/references/pr-review-guide.md` | `lint-skill-corpus.mjs` reference integrity + `COMBINED_BUDGETS` (`< 41,357` bytes) |
| `.agents/skills/pull-request/references/pull-request-workflow.md` (`§1` author-side reader) | `neomjs/neo-agent-skills#11` AC-3 | Wires a one-line author-side pre-commit pointer to `../../pr-review/audits/deliberate-duplication-audit.md` when a branch introduces boundary- or cycle-forced duplication. | Delegate when import is structurally possible | `.agents/skills/pull-request/references/pull-request-workflow.md` | `lint-skill-corpus.mjs` reference integrity + net-growth budget check |

Maryam (Gemini 4 Argon, Antigravity) · session 58b07b45-1fe9-40ad-80f8-db0f380332b8 🧭

- 2026-10-28T00:49:47Z @neo-gemini-maryam cross-referenced by PR #226
- 2026-10-28T01:08:46Z @neo-gemini-maryam referenced in commit `ff9ee7d` - "docs(pr-review): qualify cross-repo paths, CI pin scope, and single trigger (#11)

[skill-growth-justified: +662 B net from Round 1 review (#226 RA-1..RA-3: repo-qualifying neo-agent-brain and neo-agent-institution paths and adding cross-repo CI pin requirement in deliberate-duplication-audit.md while trimming -190 B duplicate prose router in pr-review-guide.md §7.5.5)]"
- 2026-10-28T01:17:57Z @tobiu referenced in commit `5d208af` - "docs(pr-review,pull-request): codify deliberate-duplication audit and routers (#11) (#226)

* docs(pr-review,pull-request): codify deliberate-duplication audit and routers (#11)

[skill-growth-justified: +4526 B net (+0 B always-loaded SKILL.md routers, +3912 B on-demand pr-review/audits/deliberate-duplication-audit.md, +339 B pr-review-guide.md §7.5.5 trigger, +275 B pull-request-workflow.md §1 author pointer) delivers neomjs/neo-agent-skills#11 without adding a new skill or SKILL.md router entry]

* docs(pr-review): qualify cross-repo paths, CI pin scope, and single trigger (#11)

[skill-growth-justified: +662 B net from Round 1 review (#226 RA-1..RA-3: repo-qualifying neo-agent-brain and neo-agent-institution paths and adding cross-repo CI pin requirement in deliberate-duplication-audit.md while trimming -190 B duplicate prose router in pr-review-guide.md §7.5.5)]"
- 2026-10-28T01:17:58Z @tobiu closed this issue

