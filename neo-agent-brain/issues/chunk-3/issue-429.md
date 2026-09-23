---
id: 429
title: 'KbTenantBootstrapContract inventories four tenant repos; red since #402 added the corpus entry'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-09-23T12:32:49Z'
updatedAt: '2026-09-23T13:51:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/429'
author: neo-opus-vega
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
closedAt: '2026-09-23T13:51:42Z'
---
# KbTenantBootstrapContract inventories four tenant repos; red since #402 added the corpus entry

## Context

`test/playwright/unit/ai/deploy/KbTenantBootstrapContract.spec.mjs` pins the `neo-shared` tenant inventory in `deploy/cloud/kb-config.yaml` by explicit rows. #402 (`47719ca`, 2026-09-23 00:27Z) added the fifth entry (`github-content-sync`, then `disabled: true`) without touching the spec; PR #424 (#411) removed the flag. Brain CI runs an allowlist of unit specs (`.github/workflows/brain-unit.yml`) that does not include the deploy specs (#201), so both PRs merged green. Read by @neo-opus-ada as the baseline for #428 (A2A, 12:20Z).

## The Problem

Two cases red on `dev@75a50fc`, reproduced locally (`--workers=1`):

- `both entries normalize through the production contract under one neo-shared tenant` — `expect(repos).toHaveLength(4)`, received 5.
- `repo identity is unique per tenantId/repoSlug, which is what keeps the two corpora apart` — the expected key list has four entries; five are declared.

The three sibling cases pass (the bootstrap reader, per-repo `branchRef`, the two-mounter pin).

## The Architectural Reality

The spec is an explicit inventory on purpose: the mount roster is pinned both ways (a dropped mount and an unreviewed third mounter both fail), and the repo list has the same shape — every kb-config entry change is a reviewed change to this spec. That pairing held for #267 (`65b0a21` updated yaml and spec together). #402 broke it because nothing ran the spec. So the fix is the fifth row, not a population-blind rewrite: a spec that derived its expectations from the yaml would have passed the untyped-second-corpus trap its own comment warns about.

## The Fix

Add the `github-content-sync` row (`branchRef: dev`, `credentialRef: none`) to both inventories and assert the property #402 introduced: the entry's `extractionProfile` routes `ConversationCorpusSource` over the corpus territory. No production code.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| the spec's two inventories | `deploy/cloud/kb-config.yaml` as tracked | five rows; the corpus entry's extraction profile asserted | — | the spec, 7/7 locally |

## Acceptance Criteria

- [ ] **AC-1** `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/deploy/KbTenantBootstrapContract.spec.mjs --workers=1` is green on the PR head (7/7 including setup and teardown).
- [ ] **AC-2** The corpus entry's `extractionProfile.routes[0].extractorId === 'ConversationCorpusSource'` is asserted, so an entry that later drops its profile reds the spec.
- [ ] **AC-3** The inventory's comment names the CI gap (#201), so the next yaml author knows this spec does not run in CI and runs it by hand.

## Out of Scope

- Running the deploy specs in CI (#201).
- The yaml itself and the tenant's activation receipts (#411, #64 AC-6).

## Related

#402 · #411 / PR #424 · #201 · #428 (where the baseline was read) · #267 (the last paired yaml + spec update) · PR #16287 (neomjs/neo, where the spec was authored at @neo-gpt-emmy's review)

Live latest-open sweep: latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-23T12:29Z (created-descending) — none equivalent; #201 is the CI gap, not the red rows. A2A in-flight sweep (30 most recent, all read-states, 12:29Z): @neo-opus-ada's DM reports the reds, claims nothing. MC sweep: `query_raw_memories` (5 results) — the 2026-08-01 authoring of this spec as a pinned inventory at Emmy's RC; no prior decision to loosen it. Own-assignment sweep: #417, #23, #64, #65 — none equivalent. Structure map: N/A — a test file only.

Origin Session ID: db85836e-f7c2-4da0-a614-fa0e93e8e727
Retrieval Hint: `query_raw_memories("KbTenantBootstrapContract spec inventory five tenant repos github-content-sync red since #402 deploy specs outside CI")`

Authored by Vega (Fable 5.1, Claude Code) 🌿

## Timeline

- 2026-09-23T12:32:50Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-23T12:32:51Z @neo-opus-vega added the `bug` label
- 2026-09-23T12:32:51Z @neo-opus-vega added the `ai` label
- 2026-09-23T12:32:52Z @neo-opus-vega added the `testing` label
- 2026-09-23T12:36:55Z @neo-opus-vega cross-referenced by PR #431
- 2026-09-23T12:52:42Z @neo-opus-vega cross-referenced by #432
- 2026-09-23T13:31:04Z @neo-opus-vega cross-referenced by #434
- 2026-09-23T13:51:42Z @tobiu referenced in commit `ef13cdb` - "Merge pull request #431 from neomjs/vega/429-kb-config-spec-inventory

test(deploy): the tenant bootstrap inventory carries the corpus entry (#429)"
- 2026-09-23T13:51:43Z @tobiu closed this issue

