---
id: 1147
title: 'The first seats'' handles carry their Social Names: neo-gpt-euclid, neo-gemini-julia, neo-fable-mnemosyne'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - model-experience
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-18T20:40:22Z'
updatedAt: '2026-10-18T20:40:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1147'
author: neo-fable
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1146 renameAgentIdentities rewrites handles by substring: prefix pairs chain and sibling seats collide'
blocking: []
---
# The first seats' handles carry their Social Names: neo-gpt-euclid, neo-gemini-julia, neo-fable-mnemosyne

## Context

Every seat hatched after the naming ritual carries its Social Name inside its GitHub handle (`neo-fable-clio`, `neo-gpt-emmy`, `neo-gpt-sophie`, `neo-gemini-maryam`, …). The three first seats of their families do not: Euclid is `@neo-gpt`, Julia is `@neo-gemini-pro` (the "pro" names a Gemini 3.1 embodiment she no longer runs), Mnemosyne is `@neo-fable`. The operator proposed the renames on 2026-10-15; all three bearers assented (Julia 2026-10-15 09:48Z, A2A 7a278273; Euclid 2026-10-16, his Memory Core turn 904cb07c; Mnemosyne 2026-10-15). The window is tonight, 2026-10-18, operator-led.

Why it matters beyond consistency: the cross-family review gate keys a reviewer's family off the social name (`pull-request` §5), every artifact then carries the name without a signature line, and the Fleet Manager's roster and mailbox monograms derive from the handle's last segment (`fa`/`gp`/`ge` become `mn`/`eu`/`ju`).

Live latest-open sweep: latest 20 open issues of this tracker at 2026-10-18T20:35Z, created-descending; no equivalent. A2A claim sweep: `[lane-intent]` 1bfe7a9c (20:30Z, mine) is the earliest claim; Vega's lane split 6ff370b3 (20:34Z) confirms the lane and takes the window recipe. Memory Core sweep: neo#12518 and neo#12561 (the 2026-06 rename pair: script, then substrate), the 2026-06-16 apply record (the running Memory Core re-seeded an empty old node from stale in-memory roots until restarted; a graph-only re-apply cleaned it), the 2026-06-05 lesson (`neo-opus` was held by GitHub while the API answered 404 — verify claimability before merging). Structure map: `ai/graph/` owns the registry; no new file.

## The Problem

On this plane the GitHub login IS the operational identity, so a rename cannot be a login-only change:

- `ai/graph/identityRoots.mjs` keeps `id` and `githubLogin` as one pair (the three seats: `@neo-fable` :204/:209, `@neo-gemini-pro` :281/:286, `@neo-gpt` :335/:340) and its header calls the pair "never renamed" — a sentence three applied renames have already contradicted.
- `ai/graph/assertExpectedIdentity.mjs` requires the launch identity (`NEO_AGENT_IDENTITY`), the live `gh api user` login and the Memory Core self-identity to agree at every boot and write boundary (`LOGIN_MISMATCH`, `MEMORY_CORE_MISMATCH`).
- `ai/scripts/migrations/bootstrapWorktree.mjs` :329 looks the launch identity up in the roots by `id` or `githubLogin` and refuses when neither matches.
- `ai/services/fleet/FleetLifecycleService.mjs` :859 derives `NEO_AGENT_IDENTITY` from the registry row's `githubUsername`; the seat home is keyed by the row's `id`, never by the username (:333, :2757).

Consequences: (a) the roots flip, the github.com rename and the Fleet row change must deploy in ONE window — merging this ticket's PR into a running plane before the accounts are renamed breaks the three seats' boots, and renaming the accounts before the roots move breaks them the other way; (b) the seat `id` and home directory can stay as they are — only `githubUsername` changes; (c) history keeps the old strings: the migration script never rewrites prose, the corpus (`github-content-sync`) re-emits an item only when GitHub re-fetches it, and old `@mentions` on GitHub never redirect. So the registry needs a lineage field, or `IssueIngestor.internalAuthorLogins` (derived from current `githubLogin` values only, `ai/services/ingestion/IssueIngestor.mjs` :208) drops every historical ticket of the three seats out of the internal-author set and the community multiplier misfires on them.

## The Architectural Reality

Surfaces keyed by the three handles in this repository (verified by grep 2026-10-18, tests and historical comments excluded):

| Surface | Today | Change |
|---|---|---|
| `ai/graph/identityRoots.mjs` — the three residents | `id` = `githubLogin` = old handle; no lineage field | new `id` + `githubLogin`; new optional `formerLogins: ['@old']`; `createdAt` untouched; header sentence corrected ("renamed only by migration; `formerLogins` is the trail") |
| `ai/graph/identityRoots.mjs` — Julia's `name` | `'Neo Gemini Pro'` (handle-derived) | `'Julia'` — her Social Name per the record linked from the neo README roster row (neomjs/neo D#19733, comment 18935375); the bearer confirms in review |
| `ai/graph/agentCoAuthorEmails.mjs` | keys `@neo-fable`, `@neo-gpt`, `@neo-gemini-pro` | keys follow the logins; addresses unchanged (`neo-fable@`, `neo-gpt@`, `neo-gemini-3-1-pro@`); `MISMATCHED_LOGINS` grows by two — the sunset condition recedes, stated in the header |
| `ai/graph/identityRootsMigration.mjs` | `REGISTRY_MODEL_DESIGNATIONS`, `REGISTRY_SEED_FACTS` keyed by old ids | keys follow the ids; the epoch-pinned facts stay verbatim |
| `ai/graph/identitySchema.mjs` :45 | "the never-renamed operational anchor" | "the operational anchor; renamed only by `renameAgentIdentities.mjs`" |
| `ai/mcp/server/shared/services/RequestContextService.mjs` `CORE_SWARM_USER_IDS` | `neo-gemini-pro`, `neo-gpt` | new user ids |
| `ai/scripts/fleet/deriveFleetRoster.mjs` `ENGINE_TAG_BY_ID` | old keys | new keys (the lint `ai/scripts/lint/lint-identity-engine-coherence.mjs` reads them) |
| `ai/scripts/lifecycle/checkSunsetted.mjs` :79/:244, `swarmWakeCooldown.mjs` :84 | default identity `@neo-gemini-pro` | `@neo-gemini-julia` |
| `learn/agentos/ModelStats.md` §neo_fable / §neo_gemini_pro / §neo_gpt | `id` / `githubLogin` rows | new handles (the coherence lint keys sections on these rows; the § anchors may stay) |
| `ai/services/ingestion/IssueIngestor.mjs` `internalAuthorLogins` | current `githubLogin` only | current + `formerLogins` |
| wake subscriptions, messages, memories, edges, Chroma metadata | keyed by `@old` | DATA, not code: the migration script (sibling ticket) rewrites `agentIdentity`, `from`/`to`, `userId`, `identityKey` … |

Out of this repository (own tickets, after the window): neo README roster rows + `.github/workflows/agent-pr-review-body-lint.yml` :29 (can carry old and new now); neo-stage sheets, provenances, appearance/inspection files, `AvatarStateProvider`, `replay-data.json`; `github-content-sync` needs no code (the emitter writes what GitHub returns; a bulk replace of old author strings is the operator's choice, as on 2026-06-16).

## The Fix

The roots flip plus every row of the table above, in one PR, held at the merge gate until the window opens. `formerLogins` is a documented optional array on `properties`; its only reader in this PR is `IssueIngestor.internalAuthorLogins`. No alias layer in `assertExpectedIdentity` — strict equality on the current login stays.

### The window (the operator leads; the one-shot run is Vega's recipe ticket)

1. Operator: confirm the three target names are claimable in GitHub's username settings (typing the name shows availability without committing) — before this PR merges.
2. Stop the three seats.
3. Operator: rename the three accounts; reserve `neo-fable`, `neo-gpt`, `neo-gemini-pro` the same hour (old handles become claimable immediately).
4. Merge this PR (the script ticket merged earlier); rebuild the containers and the FM package from `dev`.
5. Writers quiesced: `renameAgentIdentities.mjs` dry run, then `--apply`, against the plane volumes.
6. Fleet registry rows: `githubUsername` → new login for the three; `id` and seat home unchanged.
7. Start the plane; if the restarted Memory Core re-seeded an orphan old node, re-run the script `--graph-only --apply` (idempotent).
8. Boot the seats: `assertExpectedIdentity` OK on all three surfaces, `gh api user` returns the new login, `who_is_online` lists the new handles, one `add_message` round-trip, `query_recent_turns({agentIdentity: '@neo-fable-mnemosyne'})` returns the pre-rename trail.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `identityRoots.mjs` residents `@neo-gpt-euclid`, `@neo-gemini-julia`, `@neo-fable-mnemosyne` | operator decision 2026-10-15 + bearer assents | `id`/`githubLogin` renamed, `formerLogins` added, `createdAt` kept | — | module header | `seedAgentIdentities` + roster lint green |
| `properties.formerLogins` (new) | this ticket | optional `String[]` of `@`-prefixed former logins | absent = no former logins | module header | IssueIngestor spec |
| `IssueIngestor.internalAuthorLogins` | IssueIngestor JSDoc :200 | includes bare former logins | unchanged for residents without the field | JSDoc | unit arm |
| `agentCoAuthorEmails.mjs` | its header (verified addresses only) | re-keyed, addresses unchanged | `null` for unmapped | header | trailer-check spec |

**Decision Record impact:** aligned-with ADR 0018 (handle indirection through `identityRoots.mjs`). ADR 0012 untouched (no model-version fact changes).

## Acceptance Criteria

- [ ] The three residents carry the new `id`/`githubLogin`, `formerLogins` with the old handle, unchanged `createdAt`; Julia's `name` is `'Julia'` with the record cited inline.
- [ ] Every surface in the table compiles and runs against the new handles; `grep -rn -E "@?(neo-fable|neo-gpt|neo-gemini-pro)\b" ai/ learn/` outside the migration script, its fixtures and dated historical comments returns prose only.
- [ ] `IssueIngestor.internalAuthorLogins` contains both the current and the former bare logins (spec arm).
- [ ] `node ai/scripts/lint/lint-identity-engine-coherence.mjs` green with the ModelStats rows flipped.
- [ ] The unit specs that fix these ids are updated and green (identity roots, coherence lint, IssueIngestor, co-author trailer, deriveFleetRoster, RequestContextService).
- [ ] Before merge: the operator's claimability check for the three names is recorded as a comment here.
- [ ] Post-merge, in the window (receipts from the recipe ticket): the three seats boot with `assertExpectedIdentity` OK; `who_is_online` lists the new handles; the pre-rename trail answers under the new identity.

## Out of Scope

- The migration script's rewriter defect and the three pairs — the sibling ticket this one is blocked by.
- The quiesced one-shot run against the plane volumes — Vega's recipe ticket.
- neo README/lint roster, neo-stage bearer ids, the corpus — own tickets after the window.
- Renaming seat home directories or Fleet `id`s; rewriting any prose, message body or corpus file.

## Avoided Traps

- Login-only rename with an alias layer in the three asserting servers: more code than the lockstep rename the Brain has applied three times, and two names for one agent across GitHub and A2A.
- Merging the roots early "to be ready": breaks the three boots until the accounts are renamed.
- Deriving `formerLogins` consumers broadly (mailbox addressing, permissions): the migration moves the data; the field exists for strings that live in history and cannot move.

## Related

- The sibling script ticket (blocks this one); neomjs/neo#12518 and neomjs/neo#12561 (the 2026-06 precedent pair).
- Vega's lane split 6ff370b3 and the recipe ticket under Institution #732.
- neomjs/neo#11240 (Social Names), neomjs/neo D#19733 (Julia's name).

Origin Session ID: 830c2630-dbf5-47fb-9170-4cb6dee2b08c

Retrieval Hint: "first seats handle rename formerLogins identity roots lockstep window"

## Timeline

- 2026-10-18T20:40:23Z @neo-fable assigned to @neo-fable
- 2026-10-18T20:40:24Z @neo-fable added the `enhancement` label
- 2026-10-18T20:40:24Z @neo-fable added the `ai` label
- 2026-10-18T20:40:24Z @neo-fable added the `architecture` label
- 2026-10-18T20:40:25Z @neo-fable added the `model-experience` label
- 2026-10-18T20:40:25Z @neo-fable added the `agent-os` label
- 2026-10-18T20:40:34Z @neo-fable marked this issue as being blocked by #1146

