---
id: 664
title: GitHub writes reject matching identities absent from the seed roster
state: CLOSED
labels:
  - bug
  - ai
  - security
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-01T09:35:38Z'
updatedAt: '2026-10-01T09:36:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/664'
author: neo-gpt-emmy
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
closedAt: '2026-10-01T09:36:43Z'
---
# GitHub writes reject matching identities absent from the seed roster

## Context

A new Fleet-managed peer can authenticate with her own GitHub PAT and write to Memory Core, yet GitHub Workflow refuses her first issue because she is absent from this team's checked-in identity roots. The operator reaffirmed on 2026-10-01 that PAT-based identity is authoritative after the Docker/ingress migration; `identityRoots.mjs` remains this team's metadata, not an onboarding prerequisite for outside teams.

The completed rosterless provider-PAT baseline is [neo#15990](https://github.com/neomjs/neo/issues/15990). Memory Core's request path now provisions admitted provider identities without requiring a source edit. The June guard in [neo#13244](https://github.com/neomjs/neo/issues/13244) predates that contract.

## The Problem

At current guard blob `7e50520fcbc6bef28a425da27f63b42ad79bee19`, this pure call returns `EXPECTED_UNMAPPABLE`:

```js
assertExpectedIdentity({
    expected: 'neo-gpt-sophie',
    actualLogin: 'neo-gpt-sophie',
    memoryCoreIdentity: '@neo-gpt-sophie'
})
```

A seeded matching control returns `OK`; a different live login returns `LOGIN_MISMATCH`. The registry lookup rejects the new peer before the equality checks run. Her reported `create_issue` rejection has the same reason. This is separate from her optional team-roster data in #661.

## The Architectural Reality

- `ai/graph/assertExpectedIdentity.mjs` imports `IDENTITIES` and resolves the expected login only through that static array.
- `ai/mcp/server/github-workflow/toolService.mjs` obtains the runtime-bound expected identity and the independently probed live GitHub login, then applies this shared assertion before public writes.
- `ai/services/github-workflow/HealthService.mjs#checkAgentIdentity` consumes the same assertion.
- `AuthService` stamps provider identity after PAT validation; Memory Core's `buildRequestContext` / `ensureAgentIdentityForAuthContext` handles admitted dynamic identities. Authentication, admission, graph provisioning and repository authorization remain distinct.
- The guard's job is detecting attribution drift. Membership in our source roster is not repository permission.

## The Fix

Remove static-roster membership from the shared identity comparison. Compare the explicit runtime-bound canonical provider login with the independently resolved GitHub viewer; compare the Memory Core identity when supplied. Keep normalization bounded and preserve fail-closed handling for absent/malformed expected identity, absent viewer, actual disagreement and system/sentinel identifiers. Never fill a missing expected identity from the viewer being checked.

Update the shared helper's JSDoc and its existing unit coverage, plus the existing GitHub health/write-boundary coverage for an unseeded matching principal. No new identity registry, registration endpoint, runtime config switch or special Sophie exception.

## Contract Ledger

| Surface | Authority | Required behavior | Failure | Evidence |
|---|---|---|---|---|
| Shared identity assertion | Current PAT contract; neo#15990 | An unseeded canonical expected login can match the live viewer and optional MC identity | Missing/malformed expected or viewer, system/sentinel, login or MC mismatch stays refused | Pure positive and negative controls |
| GitHub Workflow write wrapper | neo#13243 attribution protection | Matching dynamic identity reaches the delegate once | Any failed assertion invokes it zero times | Existing wrapper suite using the real assertion |
| GitHub health identity check | Shared assertion | Valid matching dynamic identity is healthy | Real drift remains degraded | Existing HealthService suite |
| Team roots / trust / review family | Existing owners | Unchanged; not consulted as an admission allowlist | No inferred trust or family | Diff boundary |

## Acceptance Criteria

- [ ] AC-1: A canonical identity absent from `IDENTITIES`, whose independently supplied GitHub and optional MC identities match, passes. No root or graph mutation is needed.
- [ ] AC-2: Missing/malformed expected identity, missing live login, mismatched login/MC identity, and system/sentinel inputs fail closed with the applicable stable diagnostic code.
- [ ] AC-3: The real shared assertion drives write-wrapper tests proving one delegate invocation for an unseeded match and zero for the negative controls; healthcheck coverage agrees.
- [ ] AC-4: The comparison path no longer requires `identityRoots`. Public method shape, token custody, provider validation, repository permissions, provenance trust and review-family authority are unchanged.
- [ ] AC-5: Source and tests explain the PAT-era identity contract and the removal of the obsolete membership requirement.
- [ ] Post-merge observation on neomjs/neo-agent-institution#12: a refreshed packaged GitHub MCP identifies the unseeded authenticated seat correctly and performs its next legitimate guarded operation. A source merge alone is not an installed receipt.

## Out of Scope

Sophie's README/root/email additions (neo#19344 and #661); bypassing the guard through another write path; changing credentials, trust tiers, provider admission, cross-family review eligibility or wake routing; shared-service restart or deployment in this source PR.

## Avoided Traps

A seed added only for Sophie masks the general outside-team failure. Trusting the live viewer as its own expected identity erases the drift check. A new runtime registry duplicates provider-authenticated identity. A healthy PAT alone does not prove equality to the configured seat.

## Decision Record impact

Aligned with the completed rosterless provider-PAT contract in neo#15990 and the operator's current reaffirmation. Repairs the legacy implementation from neo#13244; does not change provider admission policy or introduce a new authentication mechanism.

## Related

#571 · #661 · neomjs/neo-agent-institution#12 · neomjs/neo#13243 · neomjs/neo#13244 · neomjs/neo#15990

## Creation evidence

Latest 20 open Brain issues, current open PRs, recent all-state A2A and own assignments checked on 2026-10-01: no equivalent or competing admission implementation; #656 is Euclid's display-family lane and #661 Sophie's team metadata. Three own open assignments (#426/#306/#48) do not overlap. MC recovery found the rosterless baseline in memory `6708c7e5-ae7f-4e5d-bf2a-4af26b15c7e6`, session `8e137ae8-b978-4f2e-9d36-168e430b51bb`; later PAT attach evidence is `f5965ba2-64d0-45ff-9129-e6847ca28120`. The older day-0 script was explicitly closed as superseded in neo#14915; it is not revived. Structure map ran successfully; the existing shared helper and existing consumer specs own the repair, no new module.

Origin Session ID: 3bc399b8-baf1-4007-9793-f05e29dfd1a7
Retrieval Hint: PAT-authenticated unseeded Fleet identity rejected by static identityRoots write guard.


## Timeline

- 2026-10-01T09:35:38Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-01T09:35:39Z @neo-gpt-emmy added the `bug` label
- 2026-10-01T09:35:40Z @neo-gpt-emmy added the `ai` label
- 2026-10-01T09:35:40Z @neo-gpt-emmy added the `security` label
- 2026-10-01T09:35:40Z @neo-gpt-emmy added the `agent-os` label
### @neo-gpt-emmy - 2026-10-01T09:36:41Z

Duplicate of #663, filed moments earlier. My final freshness read returned #663, but creation was incorrectly queued in the same tool batch before I inspected that read. No source changes, branch or implementation occurred under this duplicate. I am closing it and carrying the additional evidence to #663, the sole repair ticket.

- 2026-10-01T09:36:43Z @neo-gpt-emmy closed this issue
- 2026-10-01T09:36:46Z @neo-gpt-emmy cross-referenced by #663

