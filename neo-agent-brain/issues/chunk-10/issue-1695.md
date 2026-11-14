---
id: 1695
title: 'Enforce D-1 18:00 Europe/Berlin lock and scheduledFor hold on requestPublish'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-14T03:55:49Z'
updatedAt: '2026-11-14T03:55:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1695'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 123
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
# Enforce D-1 18:00 Europe/Berlin lock and scheduledFor hold on requestPublish

## Context

Graduated outbound publishing policy (`.agents/skills/blog-post/references/blog-authoring-guide.md` §5 and `.agents/skills/blog-post/references/outbound-post.md` §3, landed in `neomjs/neo-agent-skills#333`) requires every dated publication to lock its final text, immutable media byte digests, and cross-family review attestation by **18:00 `Europe/Berlin` (`CET`/`CEST`) on calendar day `D-1`** ahead of target publication date `D` (with urgent incident or security notices governed by explicit human operator direction, and never backdating a missed lock).

Currently, that rule exists only in prose skill guidance: `MailboxService` (`ai/services/memory-core/MailboxService.mjs`) and the Fleet publish executor (`ai/services/fleet/publishExecutor.mjs`) accept no `scheduledFor` field on `requestPublish`, cannot distinguish an undated routine post from a dated launch/campaign post, cannot reject late agent submissions/edits/attestations after `D-1 18:00 Europe/Berlin`, and cannot hold a pre-approved dated request in `'queued'` before its `scheduledFor` release window opens.

**Ownership & label note:** Self-assigned (`@neo-gemini-maryam`) as an owned ADR 0042 follow-through (`good first issue` excluded per `ticket-create` §4 because the lane is already owned with an active delivery target and amends `adr:fm-publish-gate` across `MailboxService`, `publishExecutor`, and `memory-core/openapi.yaml`).

## The Problem

Without a mechanical `scheduledFor` contract in `MailboxService` and `publishExecutor`:
1. **Late agent submissions, edits, or attestations pass admission silently:** An agent seat can submit a dated `requestPublish` (or mutate its text via `editPublishRequest` or attach `attestPublishRevision`) after `18:00 Europe/Berlin` on `D-1` — or on day `D` itself — compressing operator review time right before publication.
2. **Timezone & DST drift (`CET` vs `CEST`):** Hardcoding `UTC+01:00` breaks six months of the year (`CEST` `UTC+02:00`, plus March/October DST transition days) and naive local timestamps without `Z`/`±HH:MM` vary by host OS `TZ`.
3. **Premature batch release on `D-1` evening:** Once dated posts are locked by `D-1 18:00 Europe/Berlin`, the human operator should be able to review and click `Approve` (`approvePublishRevision`) on `D-1` evening without a batch run or `auto-approve` switch releasing a `D`-dated post a day early on `D-1`.

## The Architectural Reality

Verified on `neomjs/neo-agent-brain` `origin/dev` (`fef1702a`):
- **`ai/services/memory-core/helpers/publishDigest.mjs` (`lines 1–92`):** Shared pure deterministic module imported by both `MailboxService.mjs` (`:162`) and `publishExecutor.mjs` (`:30`) with zero Neo/service singleton dependencies. `computePublishRevisionDigest` (`:45–64`) computes the RFC 8785 SHA-256 digest over the closed 6-field content tuple `{account, channel, format, inReplyTo, media, text}`. Non-content metadata such as `hypothesis` lives in `revision.payload` outside `revisionDigest`.
- **`ai/services/memory-core/MailboxService.mjs`:**
  - `REQUEST_PUBLISH_ALLOWED_FIELDS` (`:109–119`) and `validateAndNormalizeRequestPublishPayload` (`:217–334`) enforce strict field allowlisting (`refused: unknown-field`) and secret-key rejection (`refused: secret-field`).
  - `addMessage` (`:3365`, `:3436–3473`) mints the canonical server `timestamp` (`createdAt`), validates `requestPublish`, verifies inline `attestation`, and initializes `publication.state = 'queued'`. Callers cannot supply `createdAt` or `task.revision` (`:3416–3421`), preventing backdated locks when server time is authoritative.
  - `editPublishRequest` (`:5763–5906`) is callable by either `isOriginator` (agent seat) or `isAssignee` (human operator) while `publication.state === 'queued'`, creating `revisionNumber + 1` with `approval: null, attestation: null`.
  - `attestPublishRevision` (`:5928–6014`) records later cross-family agent review attestations (`ADR 0042 §1.4` flow 2), while `approvePublishRevision` (`:6026–6119`) is restricted exclusively to `principalClass === 'human'` (`isAssignee`).
- **`ai/services/fleet/publishExecutor.mjs` (`:1030–1262`):** Runs pre-dispatch gates on `'queued'` requests before reserving `'publishing'`. Gate `5a` (`:1034–1048`) already establishes the pattern for returning a non-destructive refusal (`publicationState: 'queued'`) without transitioning the stored mailbox record to `'refused'` when a dispatch call arrives under a transient pre-condition.
- **`ai/mcp/server/memory-core/openapi.yaml` (`:6793–6835`) & `MailboxService.spec.mjs` (`:8870–8883`):** `RequestPublishPayload` sets `additionalProperties: false`, `required: [channel, account, text, media, hypothesis, format]`, and is checked by `AC-5` to ensure `JSON.stringify(schemaObj)` contains zero substrings matching `/(token|secret|password|apikey|privatekey|credential|bearer|oauth)/i`.

## The Fix

1. **`ai/services/memory-core/helpers/publishDigest.mjs`:**
   - Export `PUBLISH_LOCK_TIMEZONE = 'Europe/Berlin'` and `parsePublishSchedule(scheduledFor, timeZone = PUBLISH_LOCK_TIMEZONE)`:
     - Returns `null` when `scheduledFor` is `undefined` or `null`.
     - Accepts either an exact calendar date `'YYYY-MM-DD'` (validating real calendar dates so e.g. `'2026-02-30'` fails) or an explicit-offset ISO-8601 date-time `'YYYY-MM-DDTHH:mm(:ss(.sss))(Z|±HH:MM)'` (rejecting naive offset-less timestamps).
     - Computes `targetDate` (`YYYY-MM-DD` in `Europe/Berlin`), `earliestReleaseMs` / `earliestReleaseIso` (`00:00:00.000 Europe/Berlin` on `targetDate` for date-only values, or exact `Date.parse(scheduledFor)` for ISO date-times), and `lockCutoffMs` / `lockCutoffIso` (`18:00:00.000 Europe/Berlin` on calendar day `D-1`, resolving `CET` `UTC+01:00` and `CEST` `UTC+02:00` across DST transitions via `Intl.DateTimeFormat`).
   - Keep `computePublishRevisionDigest` strictly over `{account, channel, format, inReplyTo, media, text}` so existing 6-field digests remain 100% compatible.
2. **`ai/services/memory-core/MailboxService.mjs`:**
   - Add `'scheduledFor'` to `REQUEST_PUBLISH_ALLOWED_FIELDS` and normalize it in `validateAndNormalizeRequestPublishPayload` (`refused: scheduledFor` on invalid format/date).
   - Add an internal service clock seam `_now = () => Date.now()` (never caller-supplied on wire/MCP arguments) used to stamp server timestamps in `addMessage`, `editPublishRequest`, `attestPublishRevision`, and `approvePublishRevision`.
   - Enforce `refused: content-lock-window`:
     - In `addMessage` (`isRequestPublish`, agent-only): when `scheduledFor !== null` and server `admissionMs > schedule.lockCutoffMs`, refuse admission (`refused: content-lock-window`).
     - In `editPublishRequest`: when caller principal class is not `'human'`, refuse if target `scheduledFor` has `nowMs > nextSchedule.lockCutoffMs`, and refuse clearing an existing `scheduledFor` to `null` after `currentSchedule.lockCutoffMs` has passed (while allowing an agent to re-date `scheduledFor` forward to a future date whose `D-1 18:00 Europe/Berlin` window is still open, and allowing the human operator `principalClass === 'human'` to edit, re-date, or clear `scheduledFor` at any time while `'queued'`).
     - In `attestPublishRevision`: when the active revision has `scheduledFor !== null` and server `nowMs > schedule.lockCutoffMs`, refuse late agent attestation with `refused: content-lock-window`.
3. **`ai/services/fleet/fleetCustodyStore.mjs` & `ai/services/fleet/publishExecutor.mjs`:**
   - Add `scheduledFor: 'refused: scheduledFor'`, `contentLockWindow: 'refused: content-lock-window'`, and `scheduleNotReached: 'refused: schedule-not-reached'` to `REFUSAL_CODES`.
   - In `publishExecutor#publish` (`queued` pre-dispatch gates, re-exporting `parsePublishSchedule` and `PUBLISH_LOCK_TIMEZONE`):
     - When `payload.scheduledFor != null`, validate `parsePublishSchedule(payload.scheduledFor)` (transitioning to `'refused'` on malformed stored values).
     - Defense-in-depth lock check: when no explicit human operator `approval` is bound to `currentRevision`, transition to `'refused'` (`refused: content-lock-window`) if `Date.parse(currentRevision.createdAt) > schedule.lockCutoffMs` or (under `auto-approve after a cross-family review attestation`) `Date.parse(currentRevision.attestation?.attestedAt ?? currentRevision.attestation?.timestamp) > schedule.lockCutoffMs`.
     - Pre-schedule hold: when `nowMs < schedule.earliestReleaseMs`, return `{ ok: false, state: PUBLICATION_STATES.queued, publicationState: PUBLICATION_STATES.queued, refused: REFUSAL_CODES.scheduleNotReached, requestId, revisionDigest, scheduledFor: schedule.scheduledFor, earliestReleaseAt: schedule.earliestReleaseIso, reason }` **without** transitioning `MailboxService` out of `'queued'` and **without** calling the platform, so operator approvals recorded on `D-1` evening stay queued until `earliestReleaseAt`.
4. **`ai/mcp/server/memory-core/openapi.yaml` & `learn/agentos/decisions/0042-fm-publish-gate.md`:**
   - Add optional nullable `scheduledFor` property to `RequestPublishPayload` in `openapi.yaml` (preserving `required` and secret-free schema text).
   - Amend `learn/agentos/decisions/0042-fm-publish-gate.md` (`§1.2` and `§4.2`) to document optional `scheduledFor`, the `D-1 18:00 Europe/Berlin` lock guard (`refused: content-lock-window`), and the pre-release hold (`refused: schedule-not-reached`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `parsePublishSchedule` (`ai/services/memory-core/helpers/publishDigest.mjs`, new export) | `neomjs/neo-agent-brain#adr:fm-publish-gate` & `outbound-post.md §3` | Parses optional `'YYYY-MM-DD'` or explicit-offset ISO-8601 `scheduledFor` into `{scheduledFor, targetDate, earliestReleaseMs, earliestReleaseIso, lockCutoffMs, lockCutoffIso}` in `Europe/Berlin` (`CET`/`CEST`), leaving `computePublishRevisionDigest` unchanged. | Returns `null` when `undefined`/`null`; returns `{ok: false, code: 'refused: scheduledFor', reason}` on malformed dates, calendar rollovers (`2026-02-30`), or naive offset-less timestamps. | JSDoc + `0042-fm-publish-gate.md` | `MailboxService.spec.mjs` & `publishExecutor.spec.mjs` |
| `MailboxService` (`addMessage`, `editPublishRequest`, `attestPublishRevision` in `ai/services/memory-core/MailboxService.mjs`) | `neomjs/neo-agent-brain#adr:fm-publish-gate` §1.2, §1.4 | Admits optional `scheduledFor`; stamps authoritative server time via `_now`; enforces `serverNowMs <= lockCutoffMs` (`D-1 18:00 Europe/Berlin`) on agent `addMessage`, agent `editPublishRequest`, and `attestPublishRevision`. | Throws `refused: scheduledFor` on bad syntax or `refused: content-lock-window` past `D-1 18:00 Europe/Berlin` (or when an agent clears `scheduledFor` after lock). Human operator (`principalClass === 'human'`) retains `editPublishRequest` + `approvePublishRevision` authority at any time while `'queued'`. | `openapi.yaml` + `0042-fm-publish-gate.md` | `MailboxService.spec.mjs` (winter CET, summer CEST, DST boundary, anti-backdating, late edit/attestation, operator override) |
| `createPublishExecutor#publish` (`ai/services/fleet/publishExecutor.mjs` & `REFUSAL_CODES` in `fleetCustodyStore.mjs`) | `neomjs/neo-agent-brain#adr:fm-publish-gate` §4.2 | Holds `'queued'` dated revisions prior to `earliestReleaseMs` (`refused: schedule-not-reached`, `state: 'queued'`) and refuses unapproved late-locked revisions (`refused: content-lock-window`). | Leaves stored `MailboxService` state `'queued'` when `nowMs < earliestReleaseMs` so subsequent `publish()` at `nowMs >= earliestReleaseMs` succeeds cleanly. | JSDoc + `0042-fm-publish-gate.md` | `publishExecutor.spec.mjs` (D-1 evening pre-approval hold → day-D release, unapproved late lock refusal) |
| `RequestPublishPayload` (`ai/mcp/server/memory-core/openapi.yaml`) | `neomjs/neo-agent-brain#adr:fm-publish-gate` §1.2 | Adds optional `scheduledFor` (`type: string`, `nullable: true`) without changing `required` or bloating `task.properties`. | `null` / omitted behaves identically to current undated `requestPublish`. | `openapi.yaml` | `MailboxService.spec.mjs` (`AC-4 & AC-5`), `schemaCompaction.spec.mjs`, `McpServerListToolsSmoke.spec.mjs` |

## Decision Record Impact

`Decision Record impact: amends neomjs/neo-agent-brain#adr:fm-publish-gate`
`ADR successor-risk: adr-amendment-required — artifact new-filing/2026-11-14; ADR neomjs/neo-agent-brain#adr:fm-publish-gate (Accepted/2026-10-25); evidence learn/agentos/decisions/0042-fm-publish-gate.md + .agents/skills/blog-post/references/outbound-post.md §3; route amendment-required (updated in-band in the same PR).`

## Acceptance Criteria

- [ ] `publishDigest.mjs` exports `PUBLISH_LOCK_TIMEZONE` (`'Europe/Berlin'`) and `parsePublishSchedule`, accurately computing `D-1 18:00:00.000 Europe/Berlin` (`lockCutoffMs`) and `earliestReleaseMs` across winter `CET` (`UTC+01:00`), summer `CEST` (`UTC+02:00`), and March/October DST transition days while rejecting naive offset-less timestamps and invalid calendar dates (`'2026-02-30'`), without altering the 6-field `computePublishRevisionDigest`.
- [ ] `MailboxService` (`addMessage`, `editPublishRequest`, `attestPublishRevision`) accepts optional `scheduledFor`, uses server-owned timestamps (`_now`) so callers cannot backdate a lock, refuses late agent admissions/edits/attestations past `D-1 18:00 Europe/Berlin` (and agent attempts to strip `scheduledFor: null` post-lock) with `refused: content-lock-window`, permits agents to re-date forward into an open future window, and preserves human operator (`principalClass === 'human'`) edit and approval authority.
- [ ] `publishExecutor#publish` holds `'queued'` dated requests called before `earliestReleaseMs` (`ok: false, state: 'queued', publicationState: 'queued', refused: 'refused: schedule-not-reached'`) without calling the platform or poisoning mailbox state, releases them once `nowMs >= earliestReleaseMs`, and refuses unapproved late-locked revisions (`refused: content-lock-window`).
- [ ] `ai/mcp/server/memory-core/openapi.yaml` (`RequestPublishPayload`) and `learn/agentos/decisions/0042-fm-publish-gate.md` (`§1.2`, `§4.2`) document `scheduledFor`, `refused: content-lock-window`, and `refused: schedule-not-reached`.
- [ ] Unit tests in `MailboxService.spec.mjs`, `publishExecutor.spec.mjs`, `schemaCompaction.spec.mjs`, and `McpServerListToolsSmoke.spec.mjs`, plus `npm run agent-preflight`, pass cleanly.

## Out of Scope

- Fleet Manager UI date-picker controls (`neomjs/neo-agent-institution`).
- Automated cron/daemon polling to auto-trigger `publish()` at `earliestReleaseAt` (release remains governed by operator/executor invocation).
- Any change to the 6-field RFC 8785 `revisionDigest` tuple (`{account, channel, format, inReplyTo, media, text}`).

## Avoided Traps

- **Accepting caller-supplied `lockedAt` / `createdAt` or a seat-writable `urgentNotice: true` flag:** Either would let an agent seat backdate or self-exempt a missed `D-1 18:00` lock. Server-minted timestamps enforce the cutoff on agents, while urgent incident/security notices either omit `scheduledFor` (immediate undated queue) or use authenticated human operator `editPublishRequest` / `approvePublishRevision`.
- **Hardcoding `UTC+01:00` (`CET`) or accepting naive local timestamps:** `Europe/Berlin` observes `CEST` (`UTC+02:00`) from late March to late October; `Intl.DateTimeFormat` with `timeZone: 'Europe/Berlin'` + mandatory `Z`/`±HH:MM` offsets on date-time strings keeps every host deterministic.
- **Transitioning `MailboxService` to `'refused'` when `publish()` runs before `earliestReleaseMs`:** Would permanently block a request the operator pre-approved on `D-1` evening; holding in `'queued'` (`refused: schedule-not-reached`) mirrors Gate `5a`.
- **Including `scheduledFor` inside `computePublishRevisionDigest`:** Would break existing 6-field digest consumers (`ADR 0042 §1.4`) and invalidate content digests when a date shifts without text/media edits.

## Related

- Parent Epic: `neomjs/neo-agent-brain#123`
- Decision Record: `learn/agentos/decisions/0042-fm-publish-gate.md` (`#977`)
- Prior leaves: `#978`, `#979`, `#1271`, `#1559`, `neomjs/neo-agent-skills#333`

## Sweeps (`ticket-create` §1)

- Live latest-open + closed sweep: checked latest 20 open (`151` total open — red band, folded as sub-issue under Epic `#123` with 10 existing subs) + closed search (`requestPublish`: `#1559`, `#978`, `#1677`, `#1271`, `#977`, `#1679`, `Out of Scope` / `Avoided Traps` read on `#978` and `#1559`) at `2026-11-14T03:54Z`; no equivalent or prior exclusion found.
- A2A in-flight claim sweep: checked latest 30 messages across all read states through `2026-11-14T03:51:22Z`; zero collision.
- MC sweep: `"dated publication late lock backdate day before publish click requestPublish"`, 5 results (`995f01e4`, `e31a85ad`, etc.), confirms skill rule shipped in `neo-agent-skills#333` with mechanical Brain guard still open.
- Own-assignment sweep: `0` open in `neomjs/neo-agent-brain`, none overlapping.
- KB semantic sweep: `ask_knowledge_base` (`ticket`, 5 refs: `#19575`, `institution#673`, `#978`, `#977`), no duplicate.
- Structure map (§1c, `npm run ai:structure-map -- --files --loc`): reuses existing `ai/services/memory-core/helpers/publishDigest.mjs` (`50` LOC), `ai/services/memory-core/MailboxService.mjs`, `ai/services/fleet/fleetCustodyStore.mjs`, and `ai/services/fleet/publishExecutor.mjs` — zero new `.mjs` files created.

Origin Session ID: 5e41d607-04fd-4b94-af3b-27313b02d063
Retrieval Hint: "ADR 0042 requestPublish scheduledFor D-1 18:00 Europe/Berlin content-lock-window schedule-not-reached"

---
Authored by Maryam (Gemini 4 Argon, Antigravity). Session `5e41d607-04fd-4b94-af3b-27313b02d063`. 🧭

## Timeline

- 2026-11-14T03:55:49Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-14T03:55:50Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-14T03:55:51Z @neo-gemini-maryam added the `ai` label
- 2026-11-14T03:55:51Z @neo-gemini-maryam added the `architecture` label
- 2026-11-14T03:55:51Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-14T03:56:08Z @neo-gemini-maryam added parent issue #123
- 2026-11-14T04:12:29Z @neo-gemini-maryam cross-referenced by PR #1696
- 2026-11-14T04:48:38Z @neo-gemini-maryam referenced in commit `55737be` - "fix(publish): keep editPublishRequest attestation-free and scope schedule exports (#1695)

Co-authored-by: Maryam <neo-gemini-maryam@neomjs.com>"
- 2026-11-14T08:33:55Z @neo-opus-ada cross-referenced by #1700

