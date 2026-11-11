---
id: 1601
title: The Fleet wire reports what replacing a seat's token needs
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-11T10:58:20Z'
updatedAt: '2026-11-11T15:03:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1601'
author: neo-opus-ada
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
blocking:
  - '[ ] 902 Agent Detail replaces a stopped seat''s GitHub token'
closedAt: '2026-11-11T15:03:11Z'
---
# The Fleet wire reports what replacing a seat's token needs

## Context

neomjs/neo-agent-institution#902 lets the operator replace a stopped seat's GitHub token from Agent Detail. Its intake ([comment 6793761043](https://github.com/neomjs/neo-agent-institution/issues/902#issuecomment-6793761043), @neo-gemini-maryam) found three facts the cockpit needs that only Brain holds. This ticket puts them on the Fleet wire under one contract, so #902 consumes field names instead of guessing them.

## The Problem

1. **Which bindings a replacement may touch.** `replaceSeatCredential({id, credential, scope})` (#815) takes `'forge'` or `'forge-and-bound-plane'`. #815 requires the plane binding to be selected explicitly, and its Avoided Traps say equal token bytes are not sufficient evidence that two independently configured bindings must change together. Whether a seat holds a bound plane row is checked only inside the call, by `FleetTenantService#canPersistBoundSeatPlaneCredential`. The cockpit cannot tell whether to offer the choice, and a wrong `'forge-and-bound-plane'` comes back as `cause: 'indeterminate'` with a prose reason.
2. **Why a seat's read failed.** `openWorkProducer.mjs` classifies each failed seat read (`refused: error?.status === 401 || error?.code === 'INSUFFICIENT_SCOPES'`), then `finish()` keeps only the aggregate `reason` and `detail: failed[0]?.failure`. `projectOpenWork` (`fleetOpenWorkSource.mjs`) gives `seats[id]` only `{authored, reviewing}`. The cockpit cannot name the seat whose token was refused, or why.
3. **When a token expires.** GitHub returns `GitHub-Authentication-Token-Expiration` on responses to requests made with an expiring token. No Brain reader captures it, so a token's end arrives as a refusal.

## The Architectural Reality

- Credential custody stays in Brain. Institution ADR 0038 (`learn/agentos/decisions/0038-fm-client-topology.md`): the Body receives public projections and a session capability, never the credential. Every new field below is a non-secret value produced in Brain.
- `listAgents` and `getAgent` are registry-only (`FleetRegistryService#toPublic`, passed through by `FleetControlBridge`). Institution's roster reads `listAgents`; nothing in Institution calls `getAgent` yet.
- The typed refusal vocabulary exists in `ai/services/fleet/seatGitIdentity.mjs`: `CREDENTIAL_REFUSAL_CAUSES`, `classifyHttpCredentialRefusal`, `credentialRefusalOf`. A refused Start already carries `credentialRefusal` through `START_REFUSAL_FIELDS` (`FleetControlBridge.mjs`).
- Seat reads run with the seat's own PAT in `wireFleetOpenWorkSource.mjs` (`createGithubGraphqlQuery`; ADR 0038 §2.5.1 row 7). Institution already stamps `fleetOpenWork.seats[id]` onto roster rows (`OpenWorkRead`, `OpenWorkSeat`).
- Placement: existing modules under `ai/services/fleet/` only; no new file.

## The Fix

1. `getAgent(id)` adds `planeCredential`. The cockpit reads it when the operator acts; `listAgents` stays registry-only, so the roster poll never decrypts a store.
2. The open-work producer keeps, per seat, the typed descriptor of its latest failed read and the expiry from its latest answered read. `projectOpenWork` projects both on `seats[id]`, creating the entry for a seat that holds no rows.
3. The seat's GraphQL reads capture `GitHub-Authentication-Token-Expiration`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `getAgent(id).planeCredential` (new) | this ticket; #815 | `'none'`: the Fleet serves no plane, or its plane store holds no complete row for the seat at the endpoint `replaceSeatCredential` checks for `scope: 'forge-and-bound-plane'` (`canPersistBoundSeatPlaneCredential`). The projection and the operation share one derivation of that endpoint, so they cannot disagree. `'same-as-forge'`: a complete row whose credential equals the seat's forge credential, compared inside Brain in constant time. `'distinct'`: a complete row holding another credential. It informs the operator and never selects a scope. | `'unreadable'` when either store cannot be read; an unknown id returns `null` as today | JSDoc on `getAgent` | unit: one fixture store per state; no response carries credential bytes; a tenant-targeted seat holding a row projects the row `replaceSeatCredential` would replace |
| `fleetOpenWork` → `seats[id].credentialRefusal` (new) | this ticket; `seatGitIdentity.mjs`; [GitHub's GraphQL rate-limit docs](https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api) | The frozen `{surface, cause, verdict, status, reason}` descriptor (`credentialRefusalOf`) of the seat's latest failed read, frozen also after a reload. GraphQL `INSUFFICIENT_SCOPES` → `cause: 'insufficient-capability'`; HTTP failures go through `classifyHttpCredentialRefusal`; `reason` is the redacted failure. A rate-limited response is not a refusal: a `403` carrying GitHub's secondary-rate-limit message, `retry-after`, or `x-ratelimit-remaining: 0` is `transport-unavailable` / `unanswered`, as `429` already is. | `null` when the latest read answered; a transport failure carries `verdict: 'unanswered'`. An entry can exist while the envelope `state` is `'unavailable'`; each is bounded by its own `observedAt`, so a consumer reads it whatever the envelope state. | JSDoc on `projectOpenWork` | unit: 401, `INSUFFICIENT_SCOPES`, 5xx, a rate-limited `403`, answered, a seat with no rows |
| `fleetOpenWork` → `seats[id].tokenExpiresAt` (new) | this ticket; GitHub's `GitHub-Authentication-Token-Expiration` header | ISO-8601 UTC parsed from the header of the seat's latest answered read | `null` when the header is absent (the token has no expiry) or unparseable, never a guessed date; a failed read keeps the last answered value | JSDoc on `projectOpenWork` | unit: `UTC`, `±HHMM` and ISO-8601 fixtures; absent and unparseable → `null` |

The header's value format has no published sample: the [2021-07-26 changelog](https://github.blog/changelog/2021-07-26-expiration-options-for-personal-access-tokens/) that introduced it shows none, a docs search found none, and this seat's token has no expiry, so `gh api -i user` returned no header (2026-11-11). The parser therefore accepts `YYYY-MM-DD HH:MM:SS UTC`, `±HHMM` and ISO-8601, and projects `null` for anything else.

Ledger revised 2026-11-11 from the review of PR #1603: row 1's endpoint, row 2's rate-limit rule and envelope-state note, row 3's format provenance.

## Decision Record impact

`aligned-with` Institution ADR 0038: each new field is a public projection produced where the credential lives.

## Acceptance Criteria

- [ ] `getAgent(id).planeCredential` returns each of its four states from fixture stores, including for a tenant-targeted seat that holds a row, and no response carries credential bytes (unit).
- [ ] `listAgents` reads no credential store (unit).
- [ ] A seat whose open-work read fails with 401 or `INSUFFICIENT_SCOPES` projects its typed `credentialRefusal` on `fleetOpenWork.seats[id]`, also when it holds no rows; an answered read projects `null`; a rate-limited `403` projects `transport-unavailable`, never a refusal (unit).
- [ ] A read whose response carries the expiry header projects `tokenExpiresAt` as ISO-8601; a response without it projects `null` (unit).
- [ ] neomjs/neo-agent-institution#902's Contract Ledger names exactly these three fields (review).

## Out of Scope

- Institution rendering: neomjs/neo-agent-institution#902.
- Changing `replaceSeatCredential`. Brain never picks the scope; the operator does.
- GitLab token expiry: GitLab's API has no equivalent header.

## Avoided Traps

- `planeCredential` on `listAgents`: every roster poll would decrypt the plane store.
- Brain choosing the scope from the row or from equal bytes: #815's explicit-selection rule and its equal-bytes trap.
- Institution deriving the scope from `mcpTarget` and the plane endpoint: a plane-placed seat without a row is rejected with a prose reason.
- The cockpit parsing `reason` or `detail` text: #815 requires typed refusals.

## Related

Blocks neomjs/neo-agent-institution#902 · #815 (the operation, closed) · #1535 / #1541 (per-seat reads) · #571 (tracks #815's installed witness) · #1508 (Brain-side Fleet Manager; at 50 subs, so linked here instead of as a sub)

Live latest-open + closed sweep: latest 20 open plus all-state searches for `token expiration`, `bound plane credential`, `credential refusal open work`, `replaceSeatCredential` and `Replace token` at 2026-11-11T10:57Z. The only hit on this surface is #815 (closed); its Out of Scope and Avoided Traps were read and exclude nothing here. Backlog band: 156 open (red); no open ticket owns these fields, so this one is filed. A2A in-flight sweep: last 30 messages, no overlapping claim. Memory: the #815 / PR #1228 history and the 2026-11-09 Clio token thread. Own assignments on this surface: #571 (its comments track #815's witness as row 7) and #1508; neither holds these fields.

unowned-rationale: credential-custody work under a cross-repo contract, so not a first issue. Any maintainer with Fleet context can take it; the ledger fixes every field name.

Origin Session ID: 956d9aa9-964b-4d27-a1ce-8eb581fd8aad

Retrieval Hint: `query_raw_memories("Fleet wire planeCredential credentialRefusal tokenExpiresAt replace seat token")`

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-11T10:58:21Z @neo-opus-ada added the `enhancement` label
- 2026-11-11T10:58:22Z @neo-opus-ada added the `ai` label
- 2026-11-11T10:58:22Z @neo-opus-ada added the `agent-os` label
- 2026-11-11T10:59:55Z @neo-opus-ada cross-referenced by #902
- 2026-11-11T11:00:02Z @neo-opus-ada marked this issue as blocking #902
- 2026-11-11T12:22:59Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-11T12:36:01Z @neo-gemini-maryam cross-referenced by PR #1603
- 2026-11-11T13:41:44Z @neo-gemini-maryam referenced in commit `562a495` - "fix(fleet): share bound-plane endpoint and classify rate-limited 403s as unanswered (#1601)"
- 2026-11-11T15:03:11Z @tobiu referenced in commit `2624290` - "feat(fleet): project planeCredential, credentialRefusal and tokenExpiresAt (#1601) (#1603)

* feat(fleet): project planeCredential, credentialRefusal and tokenExpiresAt (#1601)

* fix(fleet): share bound-plane endpoint and classify rate-limited 403s as unanswered (#1601)"
- 2026-11-11T15:03:12Z @tobiu closed this issue

