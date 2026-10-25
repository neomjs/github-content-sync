---
id: 1267
title: 'Fleet publish executor: live X/LinkedIn HTTP adapters, token refresh, and bridge wiring'
state: OPEN
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-25T02:18:25Z'
updatedAt: '2026-10-25T03:12:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1267'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 19575
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 3
  signals: []
blockedBy: []
blocking: []
---
# Fleet publish executor: live X/LinkedIn HTTP adapters, token refresh, and bridge wiring

## Context

Split from `neomjs/neo-agent-brain#979` during Round-1 review of `neomjs/neo-agent-brain#1264` [QUARANTINED_URL: github.com] (`pullrequestreview-5640971354`, `RA-7` by `@neo-opus-grace`).

`neomjs/neo-agent-brain#1264` delivers the Fleet-side publish gate state machine and single-writer `MailboxService` integration in `ai/services/fleet/publishExecutor.mjs` (`publish`, `readMetrics`, `reply`, `approveRevision`, `attestRevision`, `createFleetCustodyStore`, `scrubSecrets`) over injected `platform`, `custody`, and `switchStore` seams (AC-1 through AC-6). However, three concrete runtime integration surfaces required to execute AC-7's live test-account post and read-back remain unimplemented:

1. **Live X v2 and LinkedIn Community Management HTTP platform adapters** (`publishPost`, `listRecentPosts`, `readMetrics`, `verifyOwnedPost`, plus chunked/media upload under the account's custody token).
2. **X OAuth 2.0 PKCE `offline.access` refresh-token rotation and custody persistence** (updating expired access tokens under `<userData>/brain/fleet` without requiring manual operator re-placement on every 2-hour access-token expiry).
3. **`FleetControlBridge` verb exposure and `<userData>/brain/fleet` custody directory wiring** so the Fleet Manager's operator-facing Outbound queue (`neomjs/neo#19576`) can invoke `publish(requestId, revisionDigest)`, `readMetrics(providerId)`, and `reply(...)` against the configured custody root.

- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-25T02:16:37Z` (`#1265`, `#1258`, `#1257`, `#1254`, `#1253`, `#1252`, `#1251`, `#1250`, `#1248`, `#1246`, `#1230`, `#1174`, `#1149`, `#1048`, `#1047`, `#1010`, `#1005`, `#979`, `#978`, `#969`); no duplicate leaf for the live platform client / refresh-token custody / bridge wiring found.
- **A2A in-flight claim sweep:** checked `list_messages({status: 'all', limit: 30})` at `2026-10-25T02:16:40Z`; no competing `[lane-claim]` or `[lane-intent]` on `#979` platform client / refresh / bridge scope.
- **MC sweep:** `query_raw_memories({query: 'neo-agent-brain publishExecutor X LinkedIn HTTP client offline.access refresh token FleetControlBridge custodyDir #979', nResults: 6})`, 6 results inspected, no prior declining decision found.
- **Own-assignment sweep:** 2 open in `neomjs/neo-agent-brain` (`#978`, `#979`), this ticket splits the remaining live-adapter/bridge/refresh scope from `#979` per `#1264` `RA-7`.
- **Structure map sweep:** `npm run ai:structure-map -- --files --loc` inspected `ai/services/fleet` (137 files; siblings `ai/services/fleet/publishExecutor.mjs`, `ai/services/fleet/hostEffects.mjs`, `ai/services/fleet/verifyEffect.mjs`).

## The Problem

1. **Injected `platform` seam has no production HTTP client implementation:** `ai/services/fleet/publishExecutor.mjs` calls `platform.publishPost`, `platform.listRecentPosts`, `platform.readMetrics`, and `platform.verifyOwnedPost`, which are currently satisfied only by unit-test doubles in `test/playwright/unit/ai/services/fleet/publishExecutor.spec.mjs`.
2. **X OAuth 2.0 `offline.access` access tokens expire after 2 hours:** Under ADR 0042 [QUARANTINED_URL: github.com] §2.1, X credentials use OAuth 2.0 PKCE with `tweet.read`, `tweet.write`, `users.read`, and `offline.access`. Today `createFleetCustodyStore` in `ai/services/fleet/publishExecutor.mjs` treats any `expiresAt <= nowMs` as a standing `refused: credential`. Without a refresh-token rotation path that exchanges `refreshToken` for a new `accessToken` + `refreshToken` pair and atomically persists the rotated record into `<userData>/brain/fleet`, every X account goes dark 2 hours after operator placement.
3. **No `FleetControlBridge` exposure or custody-root wiring:** Neither `FleetControlBridge` nor the Fleet service entrypoint wires `<userData>/brain/fleet` into `createPublishExecutor` or exposes `publish`, `readMetrics`, and `reply` to the Fleet Manager UI (`neomjs/neo#19576`). Consequently, `#979` AC-7 (publishing one real post to a test account and recording the provider id + read-back) has no live code path yet.

## The Architectural Reality

- `ai/services/fleet/publishExecutor.mjs` (`neomjs/neo-agent-brain#1264`) owns the 5-state dispatch table (`queued → publishing → published | refused | unresolved`), single-writer `MailboxService` transitions, revision/attestation/media byte-digest verification (`verifiedMedia: Array<{bytes: Buffer, sha256, mimeType}>`), and secret scrubbing (`scrubSecrets`).
- `ai/services/memory-core/MailboxService.mjs` (`neomjs/neo-agent-brain#1263`) owns the single-writer verbs (`transitionPublishState`, `approvePublishRevision`, `attestPublishRevision`, `editPublishRequest`) and tolerates X `[QUARANTINED_URL: t.co]` URL rewrites on read-back via `matchesRequestedPublishText`.
- `learn/agentos/decisions/0042-fleet-social-publish-gate-and-credential-custody.md` (`ADR 0042` §2.1–§2.2) specifies credential custody under `<userData>/brain/fleet`, outside seat workspaces and MCP tool schemas. Persisting a rotated X `refreshToken` / `accessToken` requires an atomic write helper in `createFleetCustodyStore` (and an explicit 1-paragraph amendment to ADR 0042 §2.2 noting that Fleet's own token-refresh rotation is the sole non-operator write to the custody file).

## The Fix

1. **X v2 & LinkedIn Community Management HTTP Adapters (`ai/services/fleet/publishExecutor.mjs` or sibling `ai/services/fleet/socialPlatformClient.mjs`):**
   - Implement `publishPost({channel, account, token, text, format, inReplyTo, media, attemptId, revisionDigest})` for X v2 (`POST /2/tweets` + media upload of `verifiedMedia[].bytes` and read-back fetch) and LinkedIn (`POST /rest/posts` with `LinkedIn-Version` header + image upload and read-back).
   - Implement `listRecentPosts({channel, account, token, limit})` returning `{id, text, createdAt}` for post-crash reconciliation bounded by `reservedAt`.
   - Implement `readMetrics({providerId, channel, account, token})` fetching X `public_metrics` + `non_public_metrics` (`impression_count`, `like_count`, `retweet_count`, `reply_count`, `quote_count`, `bookmark_count`, `url_link_clicks`, `user_profile_clicks`) and LinkedIn organization share statistics.
   - Implement `verifyOwnedPost({channel, account, providerId})` for same-account `link-reply` verification when `inReplyTo` is a raw platform `providerId`.
2. **X `offline.access` Refresh-Token Rotation & Atomic Custody Persistence:**
   - Extend `createFleetCustodyStore` to refresh expired X access tokens when `refreshToken` is present, atomically writing the rotated `{accessToken, refreshToken, expiresAt, updatedAt}` record to the custody file (`0o600` permissions) while keeping `scrubSecrets` across all error/log paths.
   - Amend `learn/agentos/decisions/0042-fleet-social-publish-gate-and-credential-custody.md` §2.2 to document Fleet-internal refresh-token rotation persistence alongside operator placement/rotation/revocation.
3. **`FleetControlBridge` Verb Exposure & Custody Wiring:**
   - Wire `<userData>/brain/fleet` custody directory and `MailboxService` into `createPublishExecutor` and expose `publish(requestId, revisionDigest)`, `readMetrics(providerId)`, and `reply(...)` on the operator-facing `FleetControlBridge` (never on any agent MCP server).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `createFleetCustodyStore#readCredential` (`ai/services/fleet/publishExecutor.mjs`) | ADR 0042 §2.1–§2.2 | When an X access token is expired and `refreshToken` (`offline.access`) is present, refreshes the token via X OAuth 2.0 token endpoint and atomically persists the rotated token pair (`0o600`) | Returns `refused: credential` (scrubbed) when refresh fails or no `refreshToken` exists | ADR 0042 §2.2 | Playwright unit test (`publishExecutor.spec.mjs`) |
| `createSocialPlatformClient` (`ai/services/fleet/publishExecutor.mjs`) | ADR 0042 §2.1, §4.2, §5 | Implements `publishPost`, `listRecentPosts`, `readMetrics`, and `verifyOwnedPost` for X v2 and LinkedIn using verified `media[].bytes` and returning `{providerId, readback, createdAt}` | Scrubbed error / `refused: platform` or `unresolved` on post-dispatch drop | JSDoc `@summary` | Playwright unit test + AC-5 live test-account receipt |
| `FleetControlBridge` (`publish`, `readMetrics`, `reply`) | ADR 0042 §1.5, §4.2 | Exposes `publish(requestId, revisionDigest)`, `readMetrics(providerId)`, and `reply(...)` bound to `MailboxService` and `<userData>/brain/fleet` to the Fleet Manager UI only | Refuses unauthenticated / non-operator calls | JSDoc `@summary` | Unit/integration test + AC-5 live post receipt |

## Decision Record impact

`amends` ADR 0042 §2.2 (`learn/agentos/decisions/0042-fleet-social-publish-gate-and-credential-custody.md`) to specify atomic custody persistence for X `offline.access` refresh-token rotation alongside operator token placement/revocation.

## Acceptance Criteria

- [ ] **AC-1:** `createSocialPlatformClient` implements `publishPost`, `listRecentPosts` (with `createdAt`), `readMetrics`, and `verifyOwnedPost` for X v2 and LinkedIn Community Management API, uploading `verifiedMedia[].bytes` (never re-reading URIs from disk).
- [ ] **AC-2:** `createFleetCustodyStore` rotates expired X access tokens using the stored `offline.access` `refreshToken`, atomically persists the rotated token pair with `0o600` permissions under `<userData>/brain/fleet`, and returns `refused: credential` (with zero secret leakage) when refresh fails; ADR 0042 §2.2 is updated to document the refresh write.
- [ ] **AC-3:** `FleetControlBridge` exposes `publish(requestId, revisionDigest)`, `readMetrics(providerId)`, and `reply(...)` wired to `MailboxService` and `<userData>/brain/fleet`, with no MCP tool exposure to agent seats.
- [ ] **AC-4:** Unit tests cover HTTP adapter request/response mapping, X refresh-token rotation + atomic file persistence + failure refusal, and bridge invocation.
- [ ] **AC-5** *(Post-merge / operator-credentialed validation from `#979` AC-7)*: One real post to a test account through `publish` is visible on the platform and recorded as `published` with its provider id and read-back in `MailboxService`.

## Out of Scope

- Fleet Manager Outbound UI cards, per-`(peer, channel)` switch selector UI, and credential placement/revocation dialogs (`neomjs/neo#19576`).
- Enabling third-party API replies before X's reply-automation rule classification is settled (`reply` remains disabled and returns `refused: reply-classification-unresolved`).

## Avoided Traps / Gold Standards Rejected

- **Re-reading media URIs inside the HTTP adapter:** Rejected because ADR 0042 §1.2 and `#1264` `RA-4` verify SHA-256 over the byte buffer before upload; the adapter must upload `media[].bytes` directly.
- **Storing refreshed tokens in SQLite or A2A messages:** Rejected per ADR 0042 §2.2; credentials and refresh tokens must remain exclusively in `<userData>/brain/fleet`.

## Related

- Parent epic: `neomjs/neo#19575`
- Sibling / predecessor ticket & PRs: `neomjs/neo-agent-brain#979`, `neomjs/neo-agent-brain#1264`, `neomjs/neo-agent-brain#978`, `neomjs/neo-agent-brain#1263`, `neomjs/neo-agent-brain#977`, `neomjs/neo-agent-brain#1261`
- Downstream UI leaf: `neomjs/neo#19576`

unowned-rationale: Filed as the `#979` / `#1264` `RA-7` split leaf while `#1263` and `#1264` complete Round-2 review convergence; ready for immediate pickup once `#1263` and `#1264` land.

Origin Session ID: e84d121d-df41-4a11-8e45-536824ab4347
Retrieval Hint: "publishExecutor RA-7 X LinkedIn HTTP client offline.access refresh token FleetControlBridge #979 #1264"


## Timeline

- 2026-10-25T02:18:26Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-25T02:18:26Z @neo-gemini-maryam added the `ai` label
- 2026-10-25T02:18:26Z @neo-gemini-maryam added the `security` label
- 2026-10-25T02:18:26Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-25T02:20:02Z @neo-gemini-maryam added parent issue #19575
- 2026-10-25T02:21:20Z @neo-gemini-maryam cross-referenced by PR #1264
- 2026-10-25T02:31:02Z @neo-gemini-maryam cross-referenced by #979
- 2026-10-25T03:12:11Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-25T03:31:51Z @neo-gemini-maryam cross-referenced by PR #1268
- 2026-10-25T04:13:34Z @neo-gemini-maryam referenced in commit `e5ed267` - "fix(fleet): isolate bridge params, single-flight refresh, and split adapters (#1267)"
- 2026-10-25T04:34:08Z @neo-gemini-maryam referenced in commit `d0f8723` - "fix(fleet): fall back to per-account custody file when bundle re-read fails (#1267)"
- 2026-10-25T04:55:08Z @neo-gemini-maryam cross-referenced by #19575

