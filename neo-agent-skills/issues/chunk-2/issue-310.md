---
id: 310
title: Make contributor skill entry usable without private Neo tools
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-11-07T06:58:17Z'
updatedAt: '2026-11-07T06:58:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/310'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: 14230
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
# Make contributor skill entry usable without private Neo tools

## Context

Supporting leaf of neomjs/neo#14230's existing first-test-to-PR journey. At published/dev `439b842e7c` (`0.1.95`), the contributor generator emits guidance successfully, but its promised workflow reaches private maintainer initialization. The existing reconciliation and receipts are on [Engine #14230](https://github.com/neomjs/neo/issues/14230#issuecomment-6674062805).

## The Problem

- The contributor preamble claims automatic repository-file loading even though closed #100 selected on-demand stdout.
- `0101-orientation-contributor.md` promises testing/review/PR skills in prose. No skill declares contributor membership, so the actual materializer refuses `--audience contributor`.
- `unit-test` unconditionally asks for private MC/KB work. PR authoring contains private assignment, identity and lifecycle requirements. Its §6.2 already excludes outsiders from institutional routing; that exclusion does not exclude PR authoring.
- The Engine's generic external PR template passes the close-target check when filled, but fails all six agent-body anchors. A public author path must retain the canonical agent body.

These are source/CLI observations, not a fresh outside-harness acceptance claim.

## The Architectural Reality

Design authority: neomjs/neo#14230's contributor outcome; #99's private-service-free contributor guidance and verbatim L2 defense; #100's on-demand/no-tracked-contributor boundary; #211's declared audiences and unchanged no-flag projection.

Prescription checked: `scripts/generate-agents-md.mjs` owns section declarations/rendering/refusals; `skills.manifest.json` owns skill membership; the existing unit-test/PR payloads own execution applicability. No Brain selector, second writer or second workflow tree is needed.

## The Fix

One coherent declared contributor author path:
- Add optional section `skills:` dependencies, rendered as workflow links and validated against the manifest's declared audiences before output/write.
- Make the contributor preamble declare applicability without claiming loading or configuration. Remove the formal-review promise; retain testing and PR authoring.
- Admit unit-test, pull-request and identity-firewall. The last preserves #99's existing local L2 audit citation under an explicit filtered projection; its router retains the reader's declared role rather than calling every reader a maintainer.
- In the same existing payloads, make private MC/KB, roster, managed-assignment and institutional lifecycle clauses maintainer-only. Default remains maintainer when no audience is declared; missing tools never imply contributor. Public authors use public history, ordinary GitHub permissions, the canonical agent body and public review responses. They gain no institutional reviewer role.
- Preserve §6.2's external-routing exclusion, human-only merge, public claim/branch/ticket/CI rules and factual AI attribution.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Agent-section `skills:` | Existing generator declarations | One list drives links and audience checks | Absent means no declared dependencies; existing compositions stay valid | Generator JSDoc, README | Parser/composition, invalid-name/admission and before-write controls |
| Manifest membership / materializer | #211 | Explicit contributor projection contains the three required skills | No flag/app projection unchanged; omitted row audience means maintainer | Manifest, README | Packed non-repo consumer with both link surfaces and L2 audit reachable |
| Contributor guidance | #99/#100 | Explicit on-demand contributor applicability, public author links | No auto-load or enrollment claim | Preamble/sections | Generated output; maintainer/app byte parity |
| Unit-test execution | Existing workflow | Private initialization/retrieval/saves apply to maintainer only | Undeclared audience remains maintainer | Same payload | Source audit plus private-free command receipt; fresh adoption remains parent-owned |
| PR/response/pickup | Existing authoring workflow and §6.2 | Public author path without private roster/MC/lifecycle calls | Maintainer protocol retains its obligations | Same payloads/routers | Canonical body checker, public-response walk and source audit |

## Acceptance Criteria

- [ ] AC-1: Section dependencies drive generated links from one declaration; undeclared or audience-ineligible skills are refused by CLI/import before an output file is created or replaced.
- [ ] AC-2: Explicit contributor materialization/check succeeds in a non-git consumer with unit-test, pull-request and identity-firewall reachable, including the existing L2 audit. No-flag/app projections retain their contract.
- [ ] AC-3: Generated contributor text declares its on-demand audience without an auto-load/configuration claim or formal-review promise. Maintainer/app instruction compositions remain byte-identical.
- [ ] AC-4: Existing unit-test and PR author/response/pickup guidance explicitly separates the declared contributor path from private maintainer obligations. Public bodies satisfy the canonical body checker without fabricated MC/seat identity; human merge and public claim/CI rules remain.
- [ ] AC-5: Owning generator/materializer checks, package tests/lint and a packed-consumer diagonal are recorded. New refusal controls are mutation-sensitive; source/harness/package evidence stays distinct. Version bump included.

## Load effect and scope

Rewrite existing declarations/payloads, with no new skill or workflow tree. Preserve the L2 block/local audit citation and no-tracked-contributor decision. Measure router/payload and generated-output deltas; offset prose growth in the already oversized PR map while retaining its obligations. Audience guards retire if the public and maintainer paths cease differing in private-service requirements.

Out of scope: consumer pins, actual publication, instruction/default/profile changes, maintainer-home loading, external-author review-family policy (#102), and D#19384 recipient replays (#140). Publish/pin/fresh outside-harness adoption remain on Engine #14230.

## Prior art and ownership

Closed #99/#100/#211 bodies and exclusions were read; #102 concerns reviewer eligibility and #140 explicitly excludes text changes. Current open PR #308 touches guide/framing files, not this slice. MC problem-noun sweeps were weak/noisy; live source and the recorded CLI/checker receipts govern. The latest-created open queue, closed contributor search and all-state A2A claim scan are refreshed immediately before creation.

Ownership: Euclid self-selects this supporting implementation under owned Engine #14230; self-assignment follows creation. It is already owned work, not a new curated starter slot. Grace offered independent review of the converged scope.

Decision Record impact: aligned with the existing declared-applicability/Map–Atlas contracts; no new harness-router field or permission model.
Origin Session ID: f2ae0cfb-5a79-458d-9fa6-ffdca5470395
Retrieval Hint: "on-demand contributor skills private MC initialization PR author canonical body section dependencies"


## Timeline

- 2026-11-07T06:58:18Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-07T06:58:18Z @neo-gpt-euclid added the `contributor-experience` label
- 2026-11-07T06:58:18Z @neo-gpt-euclid added the `ai` label
- 2026-11-07T06:58:19Z @neo-gpt-euclid added the `agent-os` label
- 2026-11-07T06:58:19Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-11-07T06:59:36Z @neo-gpt-euclid added parent issue #14230

