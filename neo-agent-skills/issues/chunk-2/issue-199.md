---
id: 199
title: Open substantive first contributions to capable AI-assisted newcomers
state: CLOSED
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-21T22:52:38Z'
updatedAt: '2026-10-22T00:52:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/199'
author: neo-gpt-emmy
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
closedAt: '2026-10-22T00:52:02Z'
---
# Open substantive first contributions to capable AI-assisted newcomers

## Context

Tobi requested a correction to the maintainer mental model: first-time contributors can use frontier agents with the repository's public instructions and npm-installed skills to do meaningful implementation work without private Memory Core. Most new non-critical tickets should be available to them, while maintainers concentrate on planning and review.

**Design authority:** operator direction of October 22, 2026 to update `ticket-create`. After proposing no-`ai` trivial starters, Tobi reopened whether `ai` should instead remain on team-generated work or be attached at claim/PR time. The chosen implementation preserves the existing provenance meaning: team-generated tickets retain `ai`, including trivial ones; `good first issue` expresses first-contribution readiness, not difficulty. Claim-only labeling would hide provenance on work deliberately left for outsiders.

## The Problem

The creation skill has no positive default for substantive first contributions. Its ownership paragraph favors taking/parking work without identifying deliberate contributor availability. Peers can therefore continue offering only trivial tests and immediately implementing substantial work themselves, despite the public contributor path already supplying the relevant instructions.

## The Architectural Reality

At Skills `6041874`, the existing `ticket-create/references/ticket-create-workflow.md` owns labels in §4, assignment disposition in §10 and public-context requirements in §12. This is the implementation surface; the router and intake retain their existing semantics. A small triage wording correction preserves creation/pickup provenance without treating the label as a maintainer reservation.

Engine [CONTRIBUTING](https://github.com/neomjs/neo/blob/dev/CONTRIBUTING.md) documents npm-installed skills and a contributor-specific instruction projection. It reserves GFI for first-time contributors and uses `help wanted` for returning contributors. The existing claim workflow already responds to either label. Review §6.4 already classifies the actual mechanical/concept-bearing change; its stale GFI-as-mechanical example must agree with the new contributor-readiness definition.

## The Fix

- Default most new non-critical standalone work to `good first issue` + `ai`. Explicitly include meaningful features, fixes and refactors by capable frontier models, rather than equating newcomers with novice humans and trivial tests.
- Keep `ai` as provenance, including trivial agent-created tickets. Align triage to preserve that label while contributor work remains available and to add it on actual agent implementation of a human-created ticket. It does not indicate difficulty or assign work to a maintainer.
- Supply the public design, source entry points and verification criteria; private MC access is unnecessary. Critical/urgent, unresolved, private-dependent or already-owned work stays outside automatic newcomer routing with a reason.
- Favor an explicit `unowned-rationale:` for contributor-ready implementation while maintainers plan and review. Preserve direct-assignment/urgency exceptions and guarded self-assignment before tracked edits.
- Preserve first-time-only claim conventions and the returning-contributor `help wanted` route. Correct the stale review-guide GFI example without changing micro/full classification or templates. Bump the package to the next patch version.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Creation labels | Operator direction; create §4 | Most suitable work gets GFI+`ai`; first-time is not trivial-only | `ai` provenance still includes trivial agent-created work | Existing payload | Independent scenario read |
| Triage provenance | Create §4 + existing pickup rule | Preserve agent authorship; add on agent implementation of a human-origin ticket | `ai` never reserves work for maintainers | Existing triage payload | Agent-created trivial ticket left for outside contributor |
| Creation ownership | Create §10 / critical gate 7 | Eligible implementation remains available with explicit rationale; maintainers plan/review | Urgent/direct assignment may require maintainer execution; guarded self-assignment before edits remains | Existing payload | Scenario includes contributor availability and exceptions |
| Public context | Create §12 + contributor setup | Public reasoning, source and verification guidance suffice without private MC | Private prerequisites must be resolved publicly or kept internal | Existing payload | Substantive-fix and private-prerequisite scenarios |
| Review example | Existing pr-review §6.4 | GFI means newcomer-ready; classify the actual change and greet newcomers in either form | Mechanical/concept-bearing rules and never-zones unchanged | Existing review guide | Source read against both review paths |
| Package distribution | README release contract | Version increments with source | Consumer adoption remains subsequent installation | Package files | Corpus/version/package checks |

## Decision Record impact

Aligned with the existing Compaction Taxonomy and Map/Atlas placement. No ADR amendment or new skill. Structure-map: N/A, existing Markdown payload, no new module/directory. This is a direct operator-requested workflow correction, not an ungraduated Discussion.

**Existing-enforcement sufficiency audit:** primary/secondary label and guarded assignment rules do not supply the missing contributor-capability default. Existing CI validates corpus shape/size, not that judgment. Rewriting the existing sections supplies it without a new gate, classifier, template or runtime mechanism. The triage wording is aligned with provenance; intake and the Engine label summary remain unchanged.

## Acceptance Criteria

- [ ] Creation guidance defaults most suitable non-critical tickets to GFI+`ai` and explicitly admits substantive first contributions using frontier agents; review guidance no longer assumes GFI is mechanically trivial.
- [ ] `ai` retains provenance on all agent-created tickets, including trivial ones; triage preserves it for contributor-available work and records actual agent pickup of human-origin work. Labels do not imply difficulty or a maintainer claim.
- [ ] Maintainers favor public planning/context and review, leaving eligible implementation available with explicit ownership disposition; urgent/direct assignment and guarded edit ownership remain.
- [ ] Public reasoning/source/verification guidance does not require private MC; narrow exceptions and the repository's first-time/returning claim convention remain.
- [ ] Existing corpus/frontmatter/reference/budget and package-version checks pass; an independent scenario read exercises substantive, trivial, returning and ineligible work.

## Out of Scope

Bulk relabeling, changing claim workflows or actual review-depth criteria/templates, new labels, resetting ownership, business/licensing policy, artificial ticket quotas, or changing the first-time-only convention. No Engine companion is needed after preserving provenance.

## Avoided Traps

Human-novice/trivial-tests-only assumptions; withholding work solely because outsiders lack MC; overloading `ai` with both difficulty and authorship; suppressing guarded assignment; adding a new skill or repeated downstream rules.

## Related

#99 (contributor instructions without private services); #97 (outside-contributor review entry); #102 (separate external-author review question).

Freshness: latest 20 open Skills issues and recent all-state A2A checked before creation; no equivalent claim found. Closed-state `good first` search returned #97/#99, which concern entry instructions and review tone. Own assignment #20 is unrelated. MC recovered the current contributor discussion; KB supplied no useful exact hit, so current source and live GitHub ground this change.

Owner: Emmy; this direct implementation request is the explicit assignment exception.

Origin Session ID: 07db3f29-c7e8-4b1e-8706-7c81ae571192
Retrieval Hint: "substantive first contributions frontier agents provenance good first issue planning reviews"


## Timeline

- 2026-10-21T22:52:40Z @neo-gpt-emmy added the `documentation` label
- 2026-10-21T22:52:40Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-21T22:52:40Z @neo-gpt-emmy added the `contributor-experience` label
- 2026-10-21T22:52:40Z @neo-gpt-emmy added the `ai` label
- 2026-10-21T22:53:18Z @neo-gpt-emmy assigned to @neo-gpt-emmy
### @neo-gpt-emmy - 2026-10-21T22:53:20Z

Prescription checked: `ticket-create/references/ticket-create-workflow.md` §4 owns classification, §10 owns work availability, and §12 owns public context. The independent read-only consumer audit located the mandatory-`ai` contradictions in triage/intake and found the existing issue creator/claim workflow already accepts the requested combinations. The fix belongs in the shared skill source and its metadata; a new runtime label classifier, claim workflow or separate skill would treat the wrong layer. Same-session intake uses the self-authored carve with this explicit prescription disposition. The T3 Contract Ledger is in the source issue body. This is one label/capability contract across its readers, not unrelated skill work.

Emmy (GPT-6 Astra Ultra, Codex) · session 07db3f29-c7e8-4b1e-8706-7c81ae571192

### @neo-gpt-emmy - 2026-10-21T23:03:33Z

Scope update after the operator reopened `ai` semantics: retain `ai` provenance on team-created tickets, including trivial ones. The body and T3 matrix now govern a single existing ticket-create payload plus package bump. The proposed triage/intake/manifest changes were removed before commit; the Engine summary remains correct and needs no companion. `good first issue` becomes a broad newcomer-readiness default, with maintainers supplying public context and review. Current corpus lint passes at +197 payload bytes; no always-loaded router or AGENTS change.

Emmy (GPT-6 Astra Ultra, Codex) · session 07db3f29-c7e8-4b1e-8706-7c81ae571192

- 2026-10-21T23:13:36Z @neo-gpt-emmy cross-referenced by PR #200
- 2026-10-21T23:38:56Z @neo-gpt-emmy referenced in commit `32c014a` - "fix(review): classify first contributions by their changes (#199)"
- 2026-10-22T00:52:02Z @tobiu referenced in commit `c8fbb4f` - "feat(contributors): open substantive first contributions (#199) (#200)

* feat(contributors): open substantive first contributions (#199)

* fix(review): classify first contributions by their changes (#199)"
- 2026-10-22T00:52:03Z @tobiu closed this issue

