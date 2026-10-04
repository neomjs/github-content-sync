---
id: 850
title: 'ADR 0041 §2.4: a provisioned profile''s declaration is the deployment''s'
state: OPEN
labels:
  - documentation
  - ai
  - architecture
assignees:
  - neo-fable-clio
createdAt: '2026-10-04T16:47:36Z'
updatedAt: '2026-10-04T16:47:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/850'
author: neo-fable-clio
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
# ADR 0041 §2.4: a provisioned profile's declaration is the deployment's

The record half of #848 (row 1 of FM v1, neomjs/neo-agent-institution#351). #848's Decision Record line reads *"`aligned-with ADR 0041` §2.4 … The record's author will add one sentence to §2.4 in a PR of her own."* ADR 0005 §6.5 keeps a decision record's revision out of the implementation PR, and a pull request resolves exactly one ticket — so the sentence needs this leaf, exactly as #842 carried §2.10 for #840. No scope is added.

## Context

#848 makes `hostLayout()` the one place the local profile's declared target (`neo-local-canonical`, its Docker-owned data root, the loopback endpoint) and its compose profiles (`cloud`, `fleet`, `ingress`) are stated; the CLI and the vessel's setup broker both read it; a run that names no target binds that declaration; the CLI's `--plane-id` / `--data-root` / `--endpoint` are overrides of it. Planner's acceptance: neomjs/neo-agent-institution#351 comments 5981994832 and 5982055393 (Sophie's correction on the profile set, 5982113503).

Live latest-open sweep: checked the latest 20 open Brain issues at 16:47Z; the only §2.4 mentions are #848 (the implementation) and #697 (cloud placement, a different clause). A2A claims: none on this scope. The sentence was first offered on PR #849 (5982117038); Mnemosyne returned it as a record revision that must travel in its own PR.

## The Problem

ADR 0041 §2 item 4 says on *Create* "the deployment declares an opaque `plane.id` before launch … that id is the run's target, and the deployment's declared data root is the run's bound root expectation" — and stops there. It does not say **who the deployment is** for a profile the host provisions, so a run the setup card starts had no target (the finding behind #848: `write-env` failed with no word, `compose-up` and `verify` answered `ok` and did nothing). The code in #849 now binds the profile's declaration; the record should say what the code does.

## The Architectural Reality

`learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`, §2 item 4 ("Binding — both comparisons"). The id keys, the root corroborates, an absent identity fails closed — all unchanged. ADR 0019 §10.7 names the canonical local plane; the compose file's health checks pin the id and root.

## The Fix

One sentence appended to §2 item 4 after *"…the deployment's declared data root is the run's bound root expectation."*:

> For a profile the host provisions, the profile's own declaration is the deployment's: the target `{planeId, dataRoot, endpoint}` and the compose profiles that bring the whole plane up are stated once in the host layout the CLI and the vessel's broker both read, and a run that names no target binds that declaration; the CLI's flags are overrides of it, never its only source (2026-10-04, #848).

Decision Record impact: `amends ADR 0041` §2.4 by one sentence — `aligned-with` the decision it already states; no rule changes.

## Acceptance Criteria

- AC-1: the sentence above is in §2 item 4 of ADR 0041 on `dev`, dated and anchored to #848; nothing else in the record changes.
- AC-2: the PR carries only this file; it merges before or beside #849, never after it is relied on silently.

## Out of Scope

Any change to the recipe, `hostLayout()`, the broker or the CLI (#848 / #849); the `local-model` profile question (none today — the local presets use the host's LM Studio).

## Related

#848 · PR #849 · #842 (the sibling record leaf for §2.10) · neomjs/neo-agent-institution#351 · neomjs/neo-agent-institution#550.

Origin Session ID: 4299144f-a074-4eee-afd9-75c53b452d15
Retrieval Hint: "ADR 0041 §2.4 provisioned profile declaration is the deployment's · hostLayout target + profiles"

## Timeline

- 2026-10-04T16:47:37Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-04T16:47:38Z @neo-fable-clio added the `documentation` label
- 2026-10-04T16:47:38Z @neo-fable-clio added the `ai` label
- 2026-10-04T16:47:38Z @neo-fable-clio added the `architecture` label
- 2026-10-04T16:50:31Z @neo-fable-clio cross-referenced by PR #852
- 2026-10-04T16:50:47Z @neo-fable-clio cross-referenced by #848
- 2026-10-04T18:16:39Z @neo-gpt cross-referenced by PR #849
- 2026-10-04T18:30:02Z @tobiu referenced in commit `f5264c5` - "docs(adr): ADR 0041 §2.4 names a provisioned profile's declaration as the deployment's (#850) (#852)"

