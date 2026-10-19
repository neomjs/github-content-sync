---
id: 191
title: 'Tests and reviews derive from what the module should do, not from what it does'
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - model-experience
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-19T03:13:55Z'
updatedAt: '2026-10-19T03:13:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/191'
author: neo-fable-mnemosyne
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
# Tests and reviews derive from what the module should do, not from what it does

## Context

Operator direction, 2026-10-19 05:0x Berlin, after the third failed Fleet Manager update in 24 h: "PR reviews and tests should follow the same paradigm: checking what the module to test SHOULD do first, and not rubber-stamp locking in something just to claim that tests got created. I am VERY sure this is not a one-off, but a common pattern." Three specimens from the same night, all green in CI, all merged:

- Brain #1156 (303bd017) asserted in a code comment that a Codex project config "wins over home for a table both name", wrote the recovery check on that premise and an arm that locks it (`launchProfileCustody.spec.mjs`, "a project table that would win over the bound home table refuses"). Codex merges config layers key by key; a ten-second `codex mcp get` would have shown it. Every Codex Desktop seat's first Start on the new FM refused (Brain #1167 / #1168).
- The same PR changed the prepared-adapter inspection's inputs and tested it with arms that never run the plane-attached Codex Desktop shape; the inspection kept demanding the plain Node env from a row that is now Fleet's launcher table (Brain #1170 / #1171). The fix PR repeated the pattern on its wiring line until review asked for the arm that runs the Start.
- The FM installer refused `handoff-incomplete` after an unclean exit, an outcome its spec asserted, although ADR 0034 §2.5's witness requires an update after an unclean exit to complete (Institution #737 / #738).

Live latest-open sweep: the 20 latest open issues here at 03:1xZ; no equivalent. Nearest: #123 (nothing enforces the `Design authority:` line), #61 (skill triggers are discipline-only and not firing), #76 (an approval survives the push that invalidates it). A2A claim sweep of the last hour: none on this scope. Memory Core rationale sweep: unavailable (embedding probe timeout 03:05Z); attested, not run.

## The Problem

A test written from the implementation proves the implementation agrees with itself. When the premise behind the implementation is wrong — a platform fact never probed, an ADR not read, a production shape not reproduced — the arm turns the bug into a contract, CI goes green, and the reviewer's §7.5 audit, which inspects "only placement and idioms", sees nothing. The review guide's §0 already puts premise authority in the substrate, not the patch, but nothing carries that rule to the moment an arm is written, and nothing asks the reviewer where each new arm's expectation came from.

## The Architectural Reality

- `pr-review-guide.md` §0 (patch-blind premise snapshot: the premise authority is the substrate), §7.5 (reviewers inspect added tests for placement and idioms only; "citation vs inference"), §9.0 (a ticket that calls an existing behavior wrong carries its `Design authority:` line).
- `unit-test/SKILL.md` + `references/unit-test.md`: authoring guidance with no rule about where an arm's expectation comes from; no distinction between a derived arm and a characterization arm.
- `ticket-create-workflow.md` §0 / `design-authority.md`: the authority line exists at ticket level; #123 shows nothing enforces it.
- CI cannot judge truth; it can only check that a claim was made.

## The Fix

One paradigm, stated once, loaded at both moments:

1. `unit-test`: before the first arm, a contract snapshot — the should-list from the authority (ADR, the owning module's JSDoc, the ticket's `Design authority:` line, or an empirical probe for a platform fact), each arm a derivation that names its source in its title or leading comment. An arm whose only source is the current implementation is a characterization arm: labeled as such, never cited as AC evidence. Arms exercise the production shape the change ships in (the plane-attached path, the installed layout), not only the function in isolation.
2. `pr-review` §7.5: a derivation line — for each added arm, which authority it derives from; an arm derived from the implementation, or a premise the ticket states without a `Design authority:` line, is a Required Action, not an inline nit. §0's "premise authority is the substrate" and §9.0's handle are the existing anchors; this is their test-side half.
3. Accretion: both edits replace text rather than add sections — §7.5's placement-and-idioms sentence becomes the derivation rule; the unit-test reference's authoring preamble becomes the contract snapshot. Net loaded bytes ≤ today's, measured with the corpus lint. Sunset: when a lint can read an arm's authority annotation, the review-side line retires to a pointer.

## Acceptance Criteria

- [ ] `unit-test` states the contract-snapshot rule and the derived-vs-characterization distinction, with the three specimens as the anchor, in no more bytes than the text it replaces.
- [ ] `pr-review-guide.md` §7.5 carries the derivation check and the Required-Action trigger; the full template's Test-Evidence audit has one line for it.
- [ ] A review of #1171's shape under the new rule names the missing wiring arm as a Required Action from the checklist alone (walked, not assumed).
- [ ] Loaded bytes of both skills do not grow (`npm run lint` corpus budget).

## Out of Scope

- A lint that reads arm annotations (follow-up once the convention exists).
- The structural complement in Brain: Ada's installed seat-journey CI (Brain #1164 / #1165).
- Enforcing the ticket-level `Design authority:` line (#123).

## Related

#123, #61, #76; neomjs/neo-agent-brain#1156, #1167, #1168, #1170, #1171; neomjs/neo-agent-institution#737, #738, #732.

Origin Session ID: 830c2630-dbf5-47fb-9170-4cb6dee2b08c
Retrieval Hint: "tests lock in the bug; should-first contract snapshot; derivation check"

## Timeline

- 2026-10-19T03:13:55Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-19T03:13:56Z @neo-fable-mnemosyne added the `testing` label
- 2026-10-19T03:13:57Z @neo-fable-mnemosyne added the `model-experience` label

