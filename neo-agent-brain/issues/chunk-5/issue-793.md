---
id: 793
title: Fleet boot throws without GH_TOKEN; the open-work source accepts none
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T07:11:52Z'
updatedAt: '2026-10-03T08:44:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/793'
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
blocking: []
closedAt: '2026-10-03T08:44:35Z'
---
# Fleet boot throws without GH_TOKEN; the open-work source accepts none

## Context

During Institution #12's installed cut at Brain `804356b` (2026-10-03 ~07:00Z), Emmy observed that the packaged Fleet Manager's Fleet child refuses to boot from Finder. It fails with `Could not authenticate with GitHub: set GH_TOKEN or GITHUB_TOKEN.` The previously installed Brain `741f9f3` booted through the same saved plane. Emmy rolled the bundle back. Until this is fixed, installed acceptance (#12) and the roster enrollment (#571) are blocked.

- **Observed (Emmy):** the error string, at boot, before any seat Start.
- **Confirmed from source (this ticket):** the throwing call site below.

## The Problem

#764 (`1b5810da`) wired the open-work producer into the Fleet entrypoint as `wireFleetOpenWorkSource({token: resolveGithubToken(), registry: FleetRegistryService})`. `resolveGithubToken` throws `github-token-unset` when neither `GH_TOKEN` nor `GITHUB_TOKEN` is set, and an app launched from Finder has neither. The source itself is built to run without a token. Its module doc says: "A server without a GitHub token still wires: every pulse answers its reason". `createGithubGraphqlQuery` refuses per call when `token` is null. So an optional source fails the whole boot eagerly.

## The Architectural Reality

- `ai/services/fleet/devFleetServer.mjs:476` (inside `boot()`): the eager call. Its comment (474–475) states the design: the token is a process secret no AiConfig leaf binds, so the entrypoint resolves it.
- `ai/services/ingestion/githubActions.mjs:30–36`: `resolveGithubToken({override, env})` returns the override, `GH_TOKEN` or `GITHUB_TOKEN`, else throws. Its other caller, `ai/scripts/maintenance/ingestCiFailures.mjs:604`, needs the refusal: CI ingest without a token is an error there.
- `ai/services/fleet/wireFleetOpenWorkSource.mjs:110` declares `options.token` as `String|null`, and `:62–64` handles null per pulse.

## The Fix

- `githubActions.mjs` gains `readGithubToken({override, env})`, which returns the token or `null`.
- `resolveGithubToken` becomes that reader plus the existing refusal: same error, same `code`. `ingestCiFailures` is unchanged.
- `devFleetServer.mjs:476` passes `readGithubToken()`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `readGithubToken({override, env})` (new export, `githubActions.mjs`) | `wireFleetOpenWorkSource`'s `String\|null` token contract | Returns override → `GH_TOKEN` → `GITHUB_TOKEN`, trimmed; `null` when none | — (it never throws) | JSDoc | AC-1 spec |
| `resolveGithubToken({override, env})` (`githubActions.mjs:30`) | The CI ingestor's need to refuse | Unchanged: the token as a `String`, else throws `code: 'github-token-unset'` | — | JSDoc | AC-2 spec |
| Fleet open-work wiring (`devFleetServer.mjs:476`) | `wireFleetOpenWorkSource.mjs:9`, `:62–64` | A null token wires the source; each pulse reads `coverage: 'unavailable'` with the missing-token reason | — | Entrypoint comment | AC-3 spec |

## Acceptance Criteria

- [ ] AC-1: `readGithubToken` returns `null` with neither variable set, and the token otherwise, with the same override → `GH_TOKEN` → `GITHUB_TOKEN` precedence as `resolveGithubToken`. Unit spec, red-first.
- [ ] AC-2: `resolveGithubToken` still throws `github-token-unset` when unset. The existing spec stays green.
- [ ] AC-3: the Fleet entrypoint passes the non-throwing reader to `wireFleetOpenWorkSource`, and the source wires with a null token (spec). A live probe of the entrypoint's expression with both variables unset is red at the base (`github-token-unset`) and `null` at the head. ~~A live receipt shows `devFleetServer` booting past the open-work wiring~~ — corrected at filing: `boot()` needs a plane or the host graph before that line, so a full boot cannot run beside the live Fleet; the full packaged boot is the Post-Merge Validation item below.

## Post-Merge Validation

- [ ] The packaged Fleet Manager boots from Finder with no GitHub token in its environment (Institution #12's re-cut, install owner).

## Out of Scope

- Supplying a token to the packaged Fleet's open-work producer: per-seat PAT routing for the open-work read is a separate design question.
- How the cockpit renders the producer's missing-token reason.

## Decision Record impact

none. GH_TOKEN stays a process secret read at the entrypoint, as #764's comment states. No AiConfig leaf is added or read.

## Related

- Introduced by #764. Consumers: #779 / #780 (open-work holder). Blocks Institution #12's installed cut and #571's enrollment.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-03T07:10:24Z, re-read 07:11:32Z; no equivalent. `#435` (closed) is the same class on a different surface (CI ingest). A2A claim sweep: 30 newest, no claim on this scope. Memory Core sweep on the symptom: no prior decision. Own-assignment sweep: no overlap. Structure map (exit 0): owning folders `ai/services/fleet`, `ai/services/ingestion`; no new file.

Retrieval Hint: "packaged Fleet boot GitHub token open-work resolveGithubToken"

Origin Session ID: 258e3158-432b-49ad-9cbe-b1568e69e7d1



## Timeline

- 2026-10-03T07:11:52Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T07:11:54Z @neo-opus-ada added the `bug` label
- 2026-10-03T07:11:54Z @neo-opus-ada added the `ai` label
- 2026-10-03T07:11:54Z @neo-opus-ada added the `regression` label
- 2026-10-03T07:11:54Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T07:17:14Z @neo-opus-ada cross-referenced by PR #794
- 2026-10-03T07:26:02Z @neo-opus-ada cross-referenced by #571
- 2026-10-03T08:44:35Z @tobiu referenced in commit `fb40366` - "fix(fleet): the Fleet boots without a GitHub token; the open-work source answers why (#793) (#794)

The entrypoint resolved the open-work producer's token with resolveGithubToken, which throws
when neither GH_TOKEN nor GITHUB_TOKEN is set, so a Finder-launched Fleet Manager refused to
boot. githubActions gains readGithubToken (the token or null); resolveGithubToken is that read
plus the existing refusal, unchanged for the CI ingestor. devFleetServer passes the reader, and
the source wires as its contract says: each pulse answers the missing token."
- 2026-10-03T08:44:35Z @tobiu closed this issue
- 2026-10-03T09:15:11Z @neo-gpt-emmy cross-referenced by PR #488

