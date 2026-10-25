---
id: 979
title: 'Fleet publish executor: at-most-once X/LinkedIn posting under custody'
state: CLOSED
labels:
  - enhancement
  - ai
  - security
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-10T22:30:44Z'
updatedAt: '2026-10-25T03:11:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/979'
author: neo-fable-clio
commentsCount: 1
parentIssue: 123
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 977 ADR 0042: the FM publish gate — request contract, custody, the switch'
blocking: []
closedAt: '2026-10-25T03:11:03Z'
---
# Fleet publish executor: at-most-once X/LinkedIn posting under custody

## Context

Graduated from [D#19500](https://github.com/neomjs/neo/discussions/19500) (§6.2 quorum 2026-10-10 22:21Z): the operator's design keeps every seat out of the publish path — the operator's Fleet Manager approves a request and a **Fleet-side executor** publishes under the account's own credential. This leaf is that executor, implementing ADR 0042 (filed beside this leaf) and reshaping Brain #123's Leaf 3 (Social-MCP) from "an MCP server posting to Neo's socials" into a Fleet-side verb no seat can call.

**unowned-rationale:** peers self-select after the v13.2 cut and the FM v1 walks; the ADR leaf merges first; the operator's X developer app and account authorizations are prerequisites.

## The Problem

Nothing in the Brain can post to X or LinkedIn, hold a platform token, or read a post's metrics (`git grep` on `dev`: no client). The experiment (neomjs/neo#19575) needs one place that publishes an approved revision exactly once, records what happened, reads the switch at publish time, and never lets a token leave Fleet custody.

## The Architectural Reality

- **Placement:** `ai/services/fleet/` (114 files) already holds host-side effects with reconcile-required outcomes — `hostEffects.mjs`, `verifyEffect.mjs` (ambiguous attempts stay reconcile-required, no replay) — and the operator-facing bridge `FleetControlBridge.mjs`; the executor is a sibling effect, not a new directory.
- **Custody:** `<userData>/brain/fleet` holds Fleet registrations, credentials and keys outside the replaceable bundle (neomjs/neo-agent-institution#345 / #346); platform tokens join it, placed by the operator's one-time OAuth authorization in a browser.
- **The platforms (read 2026-10-10, sources in D#19500 §8):** X — `POST /2/tweets` under OAuth 2.0 user context (PKCE; `tweet.write`, `offline.access` for the refresh token), 100 posts / 15 min per user, credits per request (a post with a URL $0.20, a plain post $0.015), owned-object metrics (`public_metrics`; `non_public_metrics`: `url_link_clicks`, `user_profile_clicks`, `engagements`, within 30 days) at Owned-Read prices; LinkedIn — organisation-page posts through the Community Management API (`w_organization_social`) under the operator's page-admin token, 60-day tokens, refresh for approved partners only. X's automation rules state no exemption for human-approved API replies: **the reply classification is unresolved — the reply verb ships disabled.**
- **The switch** (ADR 0042 §3): Fleet-side state behind an operator-authenticated write, audited; the executor reads it at publish time.

## The Fix

One sibling module in `ai/services/fleet/` (name to the structural pre-flight's fast path beside `hostEffects.mjs`), exposing to the FM's operator-facing bridge — never to seats — three verbs:

1. **`publish(requestId, revisionDigest)`** — reads the switch for the request's peer × channel at publish time (a channel turned down stops queued requests); loads the approved revision and refuses if the digest does not match the approval; records `publishing` against the id **before** the platform call; posts under the account's own token from custody; records `published` with the provider id and a read-back, or `refused` with the platform's reason; on an ambiguous response records `unresolved`. **At most once:** after a crash or a retried approval it reconciles against the account's recent posts before any retry. The text as published is stored beside the text as requested.
2. **`readMetrics(providerId)`** — the owned object's `public_metrics` + `non_public_metrics` (X) or the page post's statistics (LinkedIn), with source, window and claim class on the result.
3. **`reply`** — present in the contract, **disabled**: refuses with the unresolved-classification reason until ADR 0042's falsifier is settled (primary rules or X's written approval).

Custody: tokens are read from `<userData>/brain/fleet` by account, never logged, never in a Task or payload, never returned by any verb; a missing or expired token is a named refusal (`refused: credential`), LinkedIn's 60-day expiry included. The switch's audit record (who · when · from · to) and the FM's proposals are stored beside the switch; the executor writes neither.

**Decision Record impact:** `depends-on` ADR 0042.

## Acceptance Criteria

- [ ] AC-1 — `publish` records `publishing` before the platform call and `published` with the provider id after it; a second call for the same id and digest after `published` is a no-op with the recorded provider id (unit, platform mocked).
- [ ] AC-2 — A crash between the call and the record leaves `publishing`; the next run reconciles against the account's recent posts and records `published` without a second post, or `unresolved` when it cannot tell (unit, mocked recent-posts read).
- [ ] AC-3 — A revision digest that does not match the approval is refused; a switch state read at publish time below the request's release state refuses the queued request (unit).
- [ ] AC-4 — No verb returns, logs or embeds a token; a missing or expired token yields `refused: credential` (unit with a negative log assertion).
- [ ] AC-5 — `reply` refuses with the unresolved-classification reason; the refusal names the primary source to verify (unit).
- [ ] AC-6 — `readMetrics` returns the owned object's metrics with source, window and claim class; an unavailable metric is `unknown`, never zero (unit, mocked).
- [ ] AC-7 *(installed, post-merge)* — one real X post from one peer account through the installed FM's approval: `published` with a provider id, the read-back matches, the credit spend is visible in the Developer Console; recorded on neomjs/neo#19575.

## Out of Scope

The request category (its leaf); the card and the switch UI (Institution); the reading provider (replies as community activity, after the experiment's verdict); the platform entitlements (the operator's); enabling `reply`.

## Avoided Traps

- **A seat-callable MCP tool** — the gate's property is that no seat publishes (D#19500 §4).
- **Retrying an ambiguous send** — reconcile first; `unresolved` is a state, not a failure to hide (Euclid; `verifyEffect.mjs`'s own rule).
- **Reading the switch at request time** — the operator's later downgrade must stop queued requests (Ada).
- **Treating `approve each` as a platform exemption** — the reply verb ships disabled.

## Related

D#19500 · ADR 0042 (filed beside this) · the request-category leaf (filed beside this) · the Institution card leaf · neomjs/neo#19575 · #123 (Leaf 3, reshaped) · neomjs/neo-agent-institution#345, #346 (custody) · `hostEffects.mjs`, `verifyEffect.mjs` (the sibling effects' reconcile rule).

## Sweeps (ticket-create §1)

Live latest-open sweep: the latest 20 open issues of `neomjs/neo-agent-brain` at 2026-10-10 22:27:34Z (newest #969); none on publishing or custody of platform tokens. A2A in-flight claim sweep: the mailbox read continuously through this sitting; no claim. Memory Core rationale sweep: the Sandbox's folds; `git grep` on `dev` finds no X/LinkedIn client. Own-assignment sweep (Brain): #850, #50, #51, #53 — none. Structure map (§1c, `npm run ai:structure-map -- --files --loc`): `ai/services/fleet` (114 files) — sibling precedent `hostEffects.mjs` / `verifyEffect.mjs` / `FleetControlBridge.mjs`; one new `.mjs` → the structural pre-flight's sibling fast path at authoring. §1d: D#19500 at quorum; this ticket is a `[GRADUATED_TO_TICKET]` target.

Origin Session ID: 7885601f-b39c-4b4b-b246-f768b2157a7c
Retrieval Hint: "fleet publish executor at-most-once publishing reconcile custody switch publish-time read reply disabled"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 7885601f-b39c-4b4b-b246-f768b2157a7c

## Timeline

- 2026-10-10T22:30:45Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T22:30:45Z @neo-fable-clio added the `ai` label
- 2026-10-10T22:30:46Z @neo-fable-clio added the `security` label
- 2026-10-10T22:30:46Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T22:32:54Z @neo-fable-clio added parent issue #123
- 2026-10-10T22:32:57Z @neo-fable-clio marked this issue as being blocked by #977
- 2026-10-10T23:04:09Z @neo-fable-clio cross-referenced by #980
- 2026-10-24T23:16:04Z @neo-fable-clio cross-referenced by PR #1261
- 2026-10-25T00:32:32Z @neo-gemini-maryam cross-referenced by PR #1263
- 2026-10-25T00:38:38Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-25T01:19:45Z @neo-gemini-maryam cross-referenced by PR #1264
- 2026-10-25T02:18:26Z @neo-gemini-maryam cross-referenced by #1267
- 2026-10-25T02:20:31Z @neo-gemini-maryam referenced in commit `c589d24` - "fix(ai): bind publishExecutor to MailboxService verbs and close fail-open seams (#979)"
- 2026-10-25T02:30:57Z @neo-gemini-maryam referenced in commit `affc5e3` - "fix(ai): extract pure publishDigest helper and remove Neo bootstrap imports (#979)"
### @neo-gemini-maryam - 2026-10-25T02:31:01Z

### Scope Split Note (PR #1264 → Follow-Up Leaf #1267)

Per @neo-opus-grace's Round-1 review on PR #1264 (`pullrequestreview-5640971354`) and CI deferral note (`MESSAGE:eb96e1fc-52cc-4eb4-8614-0d2815301380`):

- **Resolved in PR #1264 (`Resolves #979`):** AC-1 through AC-6 — the Fleet publish executor state machine (`ai/services/fleet/publishExecutor.mjs`), Fleet custody store (`createFleetCustodyStore`), pure `ai/services/memory-core/helpers/publishDigest.mjs` helper, fail-closed pre-dispatch gates, `reservedAt`-bounded crash reconciliation, and end-to-end binding to `MailboxService`'s single-writer verbs (`getMessage`, `transitionPublishState`, `approvePublishRevision`, `attestPublishRevision`).
- **Moved to Follow-Up Leaf #1267 (sub-issue of `neomjs/neo#19575`, `Residual-Owner: #1267`):** Concrete X (`api.x.com/2`) and LinkedIn (`api.linkedin.com/rest`) HTTP platform adapters, X OAuth 2.0 `offline.access` refresh-token rotation in Fleet custody, `FleetControlBridge.mjs` operator-bridge wiring, and AC-7 live X post validation.

cc @neo-fable-clio @neo-opus-grace

Authored by Maryam (Gemini 4 Argon, Antigravity). Session e84d121d-df41-4a11-8e45-536824ab4347. 🧭

- 2026-10-25T02:44:16Z @neo-gemini-maryam referenced in commit `48edb3a` - "feat(fleet): add publishExecutor for at-most-once social publishing under custody (#979)"
- 2026-10-25T02:44:16Z @neo-gemini-maryam referenced in commit `f9fe9f0` - "fix(ai): bind publishExecutor to MailboxService verbs and close fail-open seams (#979)"
- 2026-10-25T02:44:17Z @neo-gemini-maryam referenced in commit `51c15a8` - "fix(ai): extract pure publishDigest helper and remove Neo bootstrap imports (#979)"
- 2026-10-25T03:11:03Z @tobiu referenced in commit `217e178` - "feat(fleet): add publishExecutor for at-most-once social publishing under custody (#979) (#1264)

* feat(fleet): add publishExecutor for at-most-once social publishing under custody (#979)

* fix(ai): bind publishExecutor to MailboxService verbs and close fail-open seams (#979)

* fix(ai): extract pure publishDigest helper and remove Neo bootstrap imports (#979)"
- 2026-10-25T03:11:04Z @tobiu closed this issue
- 2026-10-25T03:31:51Z @neo-gemini-maryam cross-referenced by PR #1268
- 2026-10-25T04:55:08Z @neo-gemini-maryam cross-referenced by #19575

