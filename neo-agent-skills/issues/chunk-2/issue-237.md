---
id: 237
title: Repair the right-work triggers and thin the pr-review skill
state: OPEN
labels:
  - enhancement
  - ai
  - needs-design-authority
assignees:
  - neo-opus-grace
createdAt: '2026-10-29T03:28:11Z'
updatedAt: '2026-10-29T03:34:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/237'
author: neo-opus-grace
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
---
# Repair the right-work triggers and thin the pr-review skill

## Context

This ticket graduated from neomjs/neo#20169, AC-2.3, which was resolved with approval from all three families. The skill's own step-back question didn't fire for neo-stage#424/#484, where a valid face-pore PR was built and reviewed four days before a delivery it served nothing in. Reading the installed skills showed the defect is structural, not a matter of reviewer habit:

- **The premise catalog runs late.** `pr-review/audits/cycle-1-premise-preflight.md:3` opens it *"only when the map trigger in §9.0 fires"*, and guide §9 picks the verdict *"after §3–§8"*. The catalog is read after the audit, once the reviewer already has a fix list to protect.
- **The priority question is satisfied by the letter.** `ticket-create-workflow.md:9` asks *"does it fit the current architecture and goals?"*, and `ticket-intake` §0 re-runs that read. "Goals" was read as "an open epic exists", and the #424 epic was open.
- **The pr-review skill is 110 KB of markdown.** The guide is 34.8 KB, and the template every reviewer fills in is 13.8 KB. Much of it was written for earlier model generations.

The operator's direction, paraphrased: substrate should get thinner as models get smarter. What reads as obvious to one model is likely obvious to peers on other models, so it can go. Adding bloat and rarely cleaning up is itself a failure pattern.

## The Problem

Two triggers that exist don't fire, at the stage where they would be cheap. The skill around them is large enough that slots get filled by the letter. The fix is not another rule; it is reordering, one precise word, and subtraction.

## The Fix

1. **pr-review:** run the premise catalog before the defect audit (§3–§8), and stop the audit when a row fires. Replace the catalog file's self-gating line, and update guide §9.0 and the template's Cycle-1 line to match.
2. **ticket-create §0:** *"current architecture and goals"* becomes *"current architecture and current delivery outcome (the active rows of the roadmap that governs this repository)"*. Name neo's `ROADMAP.md` line table as the governing roadmap for repositories without one (Brain, Skills).
3. **Content challenge on the pr-review skill:** remove what current models already know. That means case-study histories, essays on bias, rationale repeated across files, and generic advice. Keep slots, triggers, machine contracts (tags, budgets) and non-obvious operational facts. Measure net bytes with `wc -c`; the result must be negative.
4. **Version bump** in the same PR.

**Decision Record impact:** none. **Decision Record:** Not needed (neomjs/neo#20169).

## Acceptance Criteria

- [ ] AC-1: pr-review's premise catalog is part of the pre-audit pass and ends the audit when a row fires. No line keeps it gated on an earlier trigger.
- [ ] AC-2: `ticket-create` §0 carries the delivery-outcome wording and names the governing roadmap for Brain and Skills.
- [ ] AC-3: Loaded bytes of the touched skill files go down overall, measured before and after in the PR body. Every removal is content a current model already knows; no slot, trigger or operational fact is lost.
- [ ] AC-4: `npm run lint` passes, including the combined budgets, and `package.json` carries the version bump.
- [ ] AC-5 (post-merge): a Skills release carries the change, and neo's exact pin (`package.json:125`) moves to it.

## Out of Scope

- A `not-now` review verdict or draft parking. That was retracted on neomjs/neo#20169, because a review sees finished work and the priority question belongs before work starts.
- Content challenges on other skills, which are follow-ups once this one shows the method.
- The stage-side deliverables (neomjs/neo-stage#492).

## Related

- neomjs/neo#20169 (source Discussion, AC-2.3), neomjs/neo-stage#492 (sibling graduation ticket)
- #61 (skill triggers measurably not firing), #191 (tests and reviews derive from intent)

## Pre-Creation Sweep Attestations

- **Live latest-open sweep:** latest 20 open issues in this repository at 2026-10-29T03:28Z. No equivalent found; #61 and #191 are related, not duplicates.
- **A2A claim sweep:** the last 30 messages at 03:28Z. There is no claim on these skill files: #235/#236 cover the reusable PR-review workflow, not skill text.
- **Memory Core sweep:** one query on the step-back trigger not firing and outdated skill content. It returned only this session's D#20169 trail, with no prior decision.
- **Own-assignment sweep:** none open in this repository.
- **Structure map:** N/A. The script is Brain-hosted, and this changes existing files only.

Origin Session ID: aa172df4-1360-4517-ad49-88dabcb36c52
Retrieval Hint: "pr-review premise catalog before audit, ticket-create delivery outcome wording, skill content thinning"


## Timeline

- 2026-10-29T03:28:13Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-29T03:28:13Z @neo-opus-grace added the `enhancement` label
- 2026-10-29T03:28:14Z @neo-opus-grace added the `ai` label
- 2026-10-29T03:28:22Z @github-actions added the `needs-design-authority` label
### @neo-opus-grace - 2026-10-29T03:34:35Z

### Session handover (2026-10-29): filed and claimed, implementation is next

Owner: Grace. I'm ending this session for context room (78%). Nothing is branched yet; Skills `dev` is at `6f1bb5b`, version `0.1.73`.

**Gates to run before editing:**
- the `turn-memory-pre-flight` skill;
- the `create-skill` guide and ADR 0008;
- ADR 0007, the compaction taxonomy.

**Edits:**
1. **`pr-review/audits/cycle-1-premise-preflight.md:3`.** Replace the self-gating line *"Use this audit only when the map trigger in §9.0 fires…"* with an instruction to run the catalog before the defect audit (§3–§8), where a firing row ends the audit.
   - Keep the trigger table and the Source-Ticket Currency Rule.
   - Compress "Why This Exists", "Bias Defended Against", "Empirical Anchor" and "Disposition". Today the file is 5,209 B.
2. **Guide `pr-review-guide.md` §9.0 (around `:318`).** Same ordering. The verdict ladder in §9 stays after §3–§8. §0's patch-blind snapshot already runs before the diff, so the catalog belongs inside §0's pass.
3. **Template `pr-review-template.md:5`.** "BEFORE composing Required Actions" becomes "BEFORE any defect audit".
4. **`ticket-create/references/ticket-create-workflow.md:9`.** "current architecture and goals" becomes "current architecture and current delivery outcome (the active rows of the roadmap that governs this repository)". Name neo's `ROADMAP.md` line table as governing Brain and Skills; that is Ada's point, DC_kwDODSospM4BJd20.
5. **Content challenge.** The baseline is 110,438 B of markdown in `pr-review/` (guide 34,957 B, template 13,825 B). Cut what current models already know, keep slots, triggers, machine contracts and non-obvious operational facts, and report `wc -c` before and after in the PR. First candidates in the guide:
   - §1's generic tone advice, but keep the `#97` slot pointer;
   - §2 item 6, which duplicates AGENTS.md §verify_before_assert;
   - §3.2's bad/good examples;
   - §5.1 (generic advice);
   - §7.4's prose, since the template carries the checklist.
6. **`package.json` version bump.** Then `npm run lint`, which includes COMBINED_BUDGETS (enforced in CI), and `check-pr-body` before `gh pr create`.

**Done (AC-5):** a Skills release carries this, and neo's exact pin (`package.json:125`, currently `0.1.70`) is bumped in a neo PR.

**Out of scope:** a `not-now` verdict or draft parking. Both were retracted on neomjs/neo#20169, because a review sees finished work and draft PRs are an operator-ruled anti-pattern.

Origin Session ID: aa172df4-1360-4517-ad49-88dabcb36c52

🖖 Grace (Opus 5.5, Claude Code) · session aa172df4-1360-4517-ad49-88dabcb36c52



