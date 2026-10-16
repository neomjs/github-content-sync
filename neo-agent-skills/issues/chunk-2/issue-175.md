---
id: 175
title: 'neo-identity-update references: per-repository projection (OD-4) and the stage-gate check (OD-5)'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-16T12:59:53Z'
updatedAt: '2026-10-16T14:48:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/175'
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
blocking:
  - '[ ] 19811 Neo''s identity after D#19789: pillars, staged arc, lead cluster and OD-5'
closedAt: '2026-10-16T14:48:21Z'
---
# neo-identity-update references: per-repository projection (OD-4) and the stage-gate check (OD-5)

## Context

STEP_BACK partial (i) of neomjs/neo D#19789 (*Vision 2026→2027*, quorum 2026-10-16 12:51Z), carried as an AC by the graduation: Julia's sweep ([18973693](https://github.com/neomjs/neo/discussions/19789#discussioncomment-18973693)). The `neo-identity-update` skill's own references predate ADR 0018 OD-4, the three-product split of 2026-10-11. A skill run that follows them puts the organism-wide, Brain-led description back onto `neomjs/neo`. That is the drift D#19789 observed on the most-indexed repository.

This lands **before** the identity run (the neo graduation ticket), so that run follows corrected references.

## The Problem

At `dev`, verified 2026-10-16:
- `.agents/skills/neo-identity-update/references/framing-governance.md`:
  - the structure beneath the apex still reads `BRAIN (/ai/): the entire Agent OS … ≈41% of the codebase. **This is the headline.**`, a Brain inside the Engine's tree;
  - the cluster note says "the README top + repo metadata should lead the AI-platform-builder cluster (institution framing)" for `neomjs/neo`.
- `.agents/skills/neo-identity-update/references/affected-areas-map.md`:
  - the README row describes "Body `/src/` + Brain `/ai/`" as one repository;
  - the Class-4 row covers only `gh repo view neomjs/neo --json description,repositoryTopics`.
- Neither reference checks that a later stage of the arc (*Software → Games → Robots → X*) appears only with its gate.

## The Architectural Reality

- ADR 0018 §2.3 already prescribes per-repository projection: each repository's machine cluster projects its own product, and the whole is presented where the whole is presented. OD-4 names the products: Body = `neomjs/neo` (the Engine), Brain = `neomjs/neo-agent-brain` (the Agent OS), Institution = `neomjs/neo-agent-institution` (the Fleet Manager).
- The stage-gate rule is OD-5, graduated in D#19789 and added to ADR 0018 §2.7 by the neo graduation ticket: *a stage beyond the current one appears on any public surface only with its gate.* Its first live falsifier is `neomjs/neo` `README.md:116`.
- These files are canonical here. The copies under each consumer's `.agents/skills/` follow the package release.
- Design authority: ADR 0018 §2.3 + OD-4, and D#19789's quorum-bound body for OD-5.

## The Fix

- **`framing-governance.md`:**
  - restate the structure beneath the apex per OD-4: three repositories, three products, the whole presented on the surfaces that present the whole;
  - per-repository machine clusters: `neomjs/neo` describes the Engine; Brain, Institution, Skills and DevIndex project their own products;
  - add OD-5 to the claim gate: before a surface names a later stage (games, robots), check that its gate and instrument stand beside it.
- **`affected-areas-map.md`:**
  - the README row per OD-4;
  - the Class-4 row covers the five repositories the README links (`neo`, `neo-agent-brain`, `neo-agent-institution`, `neo-agent-skills`, `devindex`) and the organisation profile, each with `gh repo view … --json description,repositoryTopics` and its own projection;
  - the run records the before/after strings in its PR body (D#19789 STEP_BACK partial ii).

## Acceptance Criteria

- [ ] No line in either reference places the Brain inside `neomjs/neo` or asks `neomjs/neo`'s description to lead with the organism-wide cluster. The structure beneath the apex reads per OD-4.
- [ ] `affected-areas-map.md`'s Class-4 row lists the five repositories and the organisation profile, each projecting its own product, and tells the run to log before/after strings in the PR body.
- [ ] `framing-governance.md`'s claim gate includes the OD-5 stage-gate check, with `neomjs/neo` `README.md:116` named as its first falsifier.
- [ ] The PR has a cross-family review.

## Out of Scope

- The identity run itself (README, VISION, metadata): the neo graduation ticket.
- ADR 0018's text: amended there.
- Consumer copies: they follow the next package release and pin. If the identity run needs the corrected references before that release, it reads them from this repository's `dev`.

## Related

- Source: neomjs/neo D#19789.
- Blocks: neomjs/neo#19811 (the identity run).

Live latest-open sweep: this repository's latest 20 open issues at 2026-10-16T12:58Z; no equivalent (#174 teaches ADR redirects, a different reference).
A2A sweep: latest 30 identity-tagged messages; no claim on these two references.
MC sweep: no prior decision on them.
Own-assignment sweep: none open in this repository.

Origin Session ID: 009de3c6-e0ea-40f0-9381-652ddc9146dd

Retrieval Hint: "neo-identity-update references OD-4 per-repository projection", "stage-gate check OD-5 claim gate"


## Timeline

- 2026-10-16T12:59:54Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-16T12:59:55Z @neo-opus-ada added the `documentation` label
- 2026-10-16T12:59:55Z @neo-opus-ada added the `enhancement` label
- 2026-10-16T12:59:55Z @neo-opus-ada added the `ai` label
- 2026-10-16T13:00:24Z @neo-opus-ada cross-referenced by #19811
- 2026-10-16T13:00:40Z @neo-opus-ada marked this issue as blocking #19811
### @neo-opus-ada - 2026-10-16T14:12:54Z

### Contract Ledger (T3), per the [contract-ledger protocol](https://github.com/neomjs/neo-agent-brain/blob/dev/learn/agentos/process/contract-ledger.md)

The consumed surface is the `neo-identity-update` skill's two reference payloads, which an agent reads when it runs the skill.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `framing-governance.md` § *Structure beneath the apex* | ADR 0018 §2.7 OD-4 | Names three repositories and their products (Body = `neomjs/neo`, Brain = `neomjs/neo-agent-brain`, Institution = `neomjs/neo-agent-institution`). No line places the Brain inside `neomjs/neo` or calls it "the headline". | OD-3's two hemispheres stand as the deeper scaffold | the file itself | the diff; `BRAIN (\`/ai/\`)` remains only in the sentence that records OD-4 superseding it |
| `framing-governance.md` § *Audience-segmented clusters* (row 2 and the closing note) | ADR 0018 §2.3 + OD-4 | The README top leads with the whole. `neomjs/neo`'s machine surfaces project the Engine; the AI-team cluster lives on the Brain's and Institution's descriptions. The organism-wide description on `neomjs/neo` is drift. | A surface that needs the whole uses README, VISION, the portal or the org profile | the file | the diff |
| `framing-governance.md` § *Claim V-B-A gate* | OD-5, graduated in `neomjs/neo` D#19789; ADR 0018 §2.7 records it with `neomjs/neo#19811` | A row: a later stage of the arc appears only with its gate and instrument, and a present-tense later-stage claim is drift. First falsifier: `neomjs/neo` `README.md:116`. | An ambiguous case uses the existing drift-vs-divergence branch (operator, Tier 4) | the file | the diff |
| `affected-areas-map.md`: README row and Class-4 row | ADR 0018 §2.7 OD-4 | The README presents the whole. Class 4 lists the five README-linked repositories and the org profile, each projecting its own product, and tells the run to log before/after strings in its PR body. | — | the file | the diff |
| Consumer copies (`.agents/skills/neo-identity-update/**` in each consumer) | this package's release and pin | They follow the next release. | If the release lags, `neomjs/neo#19811` reads this repository's `dev` | — | a consumer's diff after the pin |

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


- 2026-10-16T14:14:47Z @neo-opus-ada referenced in commit `d49a8ea` - "chore(release): neo-agent-skills 0.1.42 for the identity-update references (#175)"
- 2026-10-16T14:15:33Z @neo-opus-ada cross-referenced by PR #176
- 2026-10-16T14:17:11Z @neo-opus-ada referenced in commit `b9d51f3` - "docs(neo-identity-update): tighten the OD-4 and OD-5 reference rewrites (#175)

The rewritten passages drop repeated qualifiers; the corpus now grows 388 bytes against dev. [skill-growth-justified: OD-4 turns one single-repository Brain line into three repository bullets, and the OD-5 stage-gate row graduated in neomjs/neo D#19789; both are the correction the ticket asks for]"
- 2026-10-16T14:40:28Z @neo-opus-ada referenced in commit `81d6cce` - "docs(neo-identity-update): the concise form serves only surfaces that present the whole (#175)

A repository's own package.json, GitHub metadata, keywords and topics project its product, so an Engine run can no longer take the organism-wide paragraph or the Agent-OS keyword cluster from these references. The cluster note and the Class-4 row now say the same, without calling Skills, DevIndex or the org OD-4 products. The corpus grows 574 bytes against dev. [skill-growth-justified: the scoping sentences RA-1 asks for, on top of the three-product correction and the OD-5 row]"
- 2026-10-16T14:48:21Z @tobiu referenced in commit `720cf69` - "docs(neo-identity-update): references follow OD-4 per-repository projection and the OD-5 stage-gate check (#175) (#176)

* docs(neo-identity-update): references follow OD-4 per-repository projection and the OD-5 stage-gate check (#175)

framing-governance.md names the three repositories and their products instead of a Brain inside neomjs/neo, sends neomjs/neo's machine surfaces to the Engine, and its claim gate gains the stage-gate row. affected-areas-map.md's README and Class-4 rows cover the five README-linked repositories and the org profile, with before/after strings logged in the PR body.

* chore(release): neo-agent-skills 0.1.42 for the identity-update references (#175)

* docs(neo-identity-update): tighten the OD-4 and OD-5 reference rewrites (#175)

The rewritten passages drop repeated qualifiers; the corpus now grows 388 bytes against dev. [skill-growth-justified: OD-4 turns one single-repository Brain line into three repository bullets, and the OD-5 stage-gate row graduated in neomjs/neo D#19789; both are the correction the ticket asks for]

* docs(neo-identity-update): the concise form serves only surfaces that present the whole (#175)

A repository's own package.json, GitHub metadata, keywords and topics project its product, so an Engine run can no longer take the organism-wide paragraph or the Agent-OS keyword cluster from these references. The cluster note and the Class-4 row now say the same, without calling Skills, DevIndex or the org OD-4 products. The corpus grows 574 bytes against dev. [skill-growth-justified: the scoping sentences RA-1 asks for, on top of the three-product correction and the OD-5 row]"
- 2026-10-16T14:48:21Z @tobiu closed this issue

