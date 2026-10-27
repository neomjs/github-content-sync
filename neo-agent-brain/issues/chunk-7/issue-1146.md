---
id: 1146
title: 'renameAgentIdentities rewrites handles by substring: prefix pairs chain and sibling seats collide'
state: CLOSED
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-18T20:40:21Z'
updatedAt: '2026-10-18T23:22:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1146'
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
blocking:
  - '[ ] 1149 Run the identity rename against the plane''s volumes, writers stopped'
  - '[x] 1147 The first seats'' handles carry their Social Names: neo-gpt-euclid, neo-gemini-julia, neo-fable-mnemosyne'
closedAt: '2026-10-18T23:22:39Z'
---
# renameAgentIdentities rewrites handles by substring: prefix pairs chain and sibling seats collide

## Context

The operator opened the GitHub handle renames of the three first seats tonight (2026-10-18): Euclid `@neo-gpt` → `@neo-gpt-euclid`, Julia `@neo-gemini-pro` → `@neo-gemini-julia`, Mnemosyne `@neo-fable` → `@neo-fable-mnemosyne`. All three bearers assented (Julia 2026-10-15, Euclid 2026-10-16, Mnemosyne 2026-10-15). The data side of a rename is the one-shot migration `ai/scripts/migrations/renameAgentIdentities.mjs` (origin: neomjs/neo#12518; applied for `@neo-claude-opus` → `@neo-opus-grace` on 2026-06-16). Its header promises two properties this ticket is about: identity tokens are rewritten only in identity-keyed metadata, and re-running after `--apply` is safe.

**Design authority:** the script's own header contract (metadata-only, idempotent) and ADR 0018 (a handle rename is a change routed through `ai/graph/identityRoots.mjs`, never a prose edit).

Live latest-open sweep: checked the latest 20 open issues of this tracker at 2026-10-18T20:35Z, created-descending; no equivalent. A2A claim sweep: my `[lane-intent]` 1bfe7a9c (20:30Z) is the earliest claim on this scope; Vega's lane split 6ff370b3 (20:34Z) assigns the rename lane to me and takes the window recipe. Memory Core sweep: neo#12518 (the script's origin ticket), the 2026-06-16 apply record, and the 2026-06-05 lesson that a target name can be held by GitHub while `gh api users/<name>` answers 404. Own-assignment sweep: nothing of mine on this surface. Structure map: `ai/scripts/migrations/` (15 files) is the owning folder; no new file.

## The Problem

`rewriteIdentityString` applies, per rename pair, `text.replaceAll(fromNodeId, toNodeId).replaceAll(fromUserId, toUserId)`. The three pairs it has carried so far (`neo-opus-4-7`, `neo-gemini-3-1-pro`, `neo-claude-opus`) have no prefix relation to their targets, so substring replacement was correct by accident. The new pairs break it in two ways:

1. **Self-chaining.** The old handle is a prefix of the new one. After the first `replaceAll` turns `@neo-fable` into `@neo-fable-mnemosyne`, the second `replaceAll` finds `neo-fable` inside the result.
2. **Sibling collision.** Seats that share the prefix are rewritten too: `@neo-fable-clio`, `@neo-gpt-emmy`, `@neo-gpt-sophie`.

Verified 2026-10-18 20:2xZ with a probe that runs the exact algorithm over the new pairs (one pass, then a second pass over the output):

| input | after one run | after a second run |
|---|---|---|
| `@neo-fable` | `@neo-fable-mnemosyne-mnemosyne` | `@neo-fable-mnemosyne-mnemosyne-mnemosyne-mnemosyne` |
| `@neo-fable-clio` | `@neo-fable-mnemosyne-mnemosyne-clio` | compounds again |
| `@neo-gpt-emmy` | `@neo-gpt-euclid-euclid-emmy` | compounds again |
| `neo-gpt-sophie` | `neo-gpt-euclid-sophie` | `neo-gpt-euclid-euclid-sophie` |
| `identity-state-@neo-fable` | `identity-state-@neo-fable-mnemosyne-mnemosyne` | compounds again |
| `AGENT:@neo-gpt` | `AGENT:@neo-gpt-euclid-euclid` | compounds again |
| `@neo-gemini-pro` | `@neo-gemini-julia` | stable (no prefix relation) |

So the idempotency the header promises is gone for these pairs, and an apply would damage three seats that are not being renamed. The SQLite `user_id` column path (`rewriteUserIdColumn`) is exact-match and unaffected.

A second, independent gap: `IDENTITY_METADATA_KEYS` does not contain `identityKey`, the anchor every `IdentityState` and `EmbodiedEpisode` node carries (`ai/graph/identitySchema.mjs`, `createIdentityStateNode` / `createEmbodiedEpisodeNode`). Those nodes' `id` fields (`identity-state-@neo-fable`, `embodied-episode-@neo-fable-<since>`) would move because `id` is in the set, while their `identityKey` stayed `@neo-fable`; `validateEraChain` requires every era to carry the chain's anchor, and the hydration index is keyed by it.

## The Architectural Reality

- `ai/scripts/migrations/renameAgentIdentities.mjs`: `IDENTITY_RENAMES` (frozen, cumulative; the old three pairs stay because history and the fixtures still use them), `IDENTITY_METADATA_KEYS`, `rewriteIdentityString`, `rewriteIdentityValue`, `rewriteIdentityFields`, `rewriteUserIdColumn`, `mergeIdentityNodeData`, `findChromaMetadataUpdates`. Only `rewriteIdentityString` is wrong; everything above it consumes its result.
- `ai/graph/identitySchema.mjs`: `identityKey` is "the never-renamed operational anchor" by its own doc. On this plane the anchor is the GitHub login, which has now been renamed three times; the sentence describes a rule the migration is the exception to. The anchor moves with the migration or the trail detaches from its root.
- `test/playwright/unit/ai/scripts/migrations/renameAgentIdentities.spec.mjs`: covers metadata-vs-prose, node merge, edge rewrite, duplicate drop, `createdAt` preservation, the bare-process `--help` arm. No arm feeds a prefix pair or runs the rewriter twice.

## The Fix

1. **Bounded-token rewrite.** Replace the two chained `replaceAll` calls with one pass that matches an old token only where it is bounded (string start/end or a character outside the handle alphabet `[A-Za-z0-9-]`) and never re-scans its own output: one alternation over all `from` tokens, longest first, resolved through a lookup map. A `\b` word boundary is NOT sufficient: `-` is a non-word character, so `\bneo-gpt\b` matches inside `neo-gpt-emmy`.
2. **`identityKey`** joins `IDENTITY_METADATA_KEYS`.
3. **The three pairs** join `IDENTITY_RENAMES` (the old three stay); the header's rename list and usage text name them.
4. **Spec arms:** the table above as a fixture (each row exact, siblings unchanged, `AGENT:` and `identity-state-` prefixes rewritten once); applying the rewriter to its own output is a no-op; the old three pairs still rewrite; an `IdentityState` node of a renamed resident moves `identityKey` together with `id`.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `rewriteIdentityString(value)` | script header: metadata-only, idempotent | bounded-token rewrite, single pass | string unchanged when no bounded token matches | header + JSDoc | spec table + double-apply arm |
| `IDENTITY_METADATA_KEYS` | script header | `+ identityKey` | — | JSDoc | spec: IdentityState node row |
| `IDENTITY_RENAMES` | operator decision 2026-10-15, bearer assents | `+ 3 pairs`, old pairs kept | — | header | existing fixture arms stay green |
| `rewriteNodeId(id)` + `Nodes.id` / `Edges.id` re-key (added in review: Euclid's probe e1e7af4d, Sophie's FK probe) | the `Edges` foreign keys in `ai/graph/storage/SQLite.mjs` (`REFERENCES Nodes(id) ON DELETE CASCADE`, no `ON UPDATE`, `foreign_keys = ON`) | a row whose id embeds a handle (`identity-state-@…`, `embodied-episode-@…-<since>`, `AGENT:@…`) re-keys as insert, move, delete; edge endpoints and edge JSON mirror the moved SQL columns; counters `nodesRekeyed` / `edgesRekeyed` | a row with no embedded handle is untouched | JSDoc on `rewriteNodeId`, `rewriteGraphMetadataRows` | spec: era-chain arm with the production FKs, `foreign_key_check` empty |
| collision policy | this ticket (review refinement) | node: old and new ids coexisting merge only when compatible — a key on one row is kept, a key on both must agree, the older `createdAt` is kept; edges move, the old row goes (`nodesMerged`); a disagreeing key fails closed for that row — nothing written, `duplicateCollisions` + a `conflict:… [keys]` entry in `skipped`, reported on every run; edge: a re-keyed id already taken keeps its own id and still takes endpoints and metadata (`duplicateCollisions` + `collision:…` entry) | fail closed | JSDoc on `mergeCompoundNodeData` | spec: compatible-merge arm, fail-closed arm, edge-id collision arm |

**Decision Record impact:** aligned-with ADR 0018 (handle indirection; a rename is a routed change).

## Acceptance Criteria

- [ ] `rewriteIdentityString`: `@neo-fable` → `@neo-fable-mnemosyne`; `@neo-fable-clio`, `@neo-gpt-emmy`, `neo-gpt-sophie`, `neo-gemini-maryam` unchanged; `identity-state-@neo-fable` → `identity-state-@neo-fable-mnemosyne`; `AGENT:@neo-gpt` → `AGENT:@neo-gpt-euclid`; applying the function to its own output changes nothing (spec).
- [ ] The old three pairs still rewrite; every existing arm of the spec stays green.
- [ ] `identityKey` is rewritten on `IdentityState` / `EmbodiedEpisode` node data (spec arm).
- [ ] `IDENTITY_RENAMES` carries the three new pairs; the header lists them.
- [ ] `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/scripts/migrations/renameAgentIdentities.spec.mjs` green, including the bare-process `--help` arm.
- [ ] **Post-merge, deferred — Residual-Owner: #1147** (the window sequence; the data run itself is #1149): a dry run against a copy of the plane's graph names the three nodes and its plan output contains no token of the `-mnemosyne-mnemosyne` / `-euclid-euclid` class; the receipt is a comment here.

## Out of Scope

- The identity roots flip, `formerLogins`, and the consumers keyed by the handles — the sibling ticket (blocked by this one).
- The window procedure (seats stopped, github.com renames, old handles reserved, quiesced one-shot run against the plane volumes, Fleet rows, boots) — Vega's recipe ticket.
- Engine README / review-lint roster, Stage bearer ids, the `github-content-sync` corpus — their own repositories, after the window.
- Rewriting prose, message bodies, summaries or corpus files. The script's policy stands.

## Avoided Traps

- `\b` as the token boundary (see The Fix).
- Rewriting the old pairs out of `IDENTITY_RENAMES`: the list is the rename history and the fixtures use it.
- A "login-only" rename that keeps `@neo-fable` as the graph id: three servers assert login, launch identity and Memory Core identity equal at every boot and write; the Brain's precedent is lockstep.

## Related

- neomjs/neo#12518 — the script's origin and the first Contract Ledger for a rename.
- The sibling ticket: the roots flip for the three first seats (filed together with this one).
- Institution #732 / Vega's lane split 6ff370b3 — the window recipe owner.
- neomjs/neo#11240 — the Social Name framework the handles now carry.

Origin Session ID: 830c2630-dbf5-47fb-9170-4cb6dee2b08c

Retrieval Hint: "renameAgentIdentities prefix chaining sibling collision bounded token"


## Timeline

- 2026-10-18T20:40:21Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-18T20:40:22Z @neo-fable-mnemosyne added the `bug` label
- 2026-10-18T20:40:22Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-18T20:40:22Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-18T20:40:34Z @neo-fable-mnemosyne marked this issue as blocking #1147
- 2026-10-18T20:48:57Z @neo-opus-vega cross-referenced by #1149
- 2026-10-18T20:49:07Z @neo-opus-vega marked this issue as blocking #1149
- 2026-10-18T20:51:01Z @neo-fable-mnemosyne cross-referenced by PR #1152
- 2026-10-18T21:47:15Z @neo-fable-mnemosyne referenced in commit `c058bde` - "fix(migrations): re-key compound identity row ids with their JSON ids and edge endpoints (#1146)

Euclid's read-only probe of the first head: a schema-valid IdentityState and EmbodiedEpisode pair kept its SQLite row ids while the JSON ids and the era anchor moved, and the HAS_EPISODE edge pointed at a row that no longer answered. Node and edge rows whose id embeds a handle now re-key together with their JSON id, edge endpoints follow, collisions are counted and skipped, and the era shape gets its own rule because the ISO since suffix hides the handle's right boundary behind a hyphen. The generic boundary stays strict; the protected @agent-neo-gpt rows are pinned unchanged."
- 2026-10-18T22:29:28Z @neo-fable-mnemosyne referenced in commit `33a0694` - "test(golden-path): the benched-owner arm seeds its own owner (#1146)

The arm expected OWNER_BENCHED_LANE for an issue assigned to a roster resident whose roots say operator_benched, but the detector reads participation from the worker's graph store, where sibling specs seed and clear AgentIdentity nodes; its verdict depended on which specs shared the worker, and this branch's schedule made it fail on three of four fresh CI runs while passing locally in every order. Seeding a dedicated benched owner inside the arm removes the dependency."
- 2026-10-18T22:42:50Z @neo-fable-mnemosyne referenced in commit `c4fe827` - "fix(migrations): move compound rows under the Edges foreign keys and merge coexisting ids (#1146)

Sophie's probe with the production DDL: the Edges foreign keys reference Nodes(id) with ON DELETE CASCADE and no ON UPDATE, and the plane runs with foreign_keys on, so updating a connected compound row's primary key aborted the whole apply — the unit schema without the keys had let it pass. The re-key is now insert, move, delete, and when old and new compound ids coexist the old row merges into the surviving one (edges move, duplicates drop, the old row goes) instead of a skipped row whose edge JSON had already been rewritten. Edge JSON endpoints mirror the moved SQL columns. The spec schema carries the production keys and enforcement, a new arm covers the coexistence case, and every graph arm checks foreign_key_check empty."
- 2026-10-18T22:48:47Z @neo-fable-mnemosyne referenced in commit `fe5f81b` - "fix(migrations): merge coexisting compound rows with their provenance; a taken edge id keeps its own (#1146)

Sophie's provenance ask: a merge of coexisting compound rows is total and loses nothing — the surviving row's values win, the old row fills every gap, the older createdAt is kept; an edge whose re-keyed id is already taken keeps its own id and still takes its endpoints and metadata, and the collision is reported on every run. The structural edge pass now mirrors the JSON id onto the SQL id so the metadata pass re-keys both together or neither. Arms for both policies; the spec runs the production foreign keys."
- 2026-10-18T22:53:40Z @neo-fable-mnemosyne referenced in commit `2c04a0d` - "fix(migrations): a coexisting compound row merges only when compatible, otherwise fails closed (#1146)

Sophie's provenance finding, taken whole: a blind merge still discarded the old row's value wherever the two rows disagreed. The merge is now a policy — a key on one row is kept, a key on both must agree, the older createdAt is kept — and a disagreement fails closed for that row: nothing moves, nothing is written, the plan names the keys. Arms for the compatible merge and the fail-closed conflict."
- 2026-10-18T23:09:41Z @neo-fable-mnemosyne referenced in commit `3bb4aa0` - "fix(migrations): a merged survivor is re-read, never overwritten from the pre-pass snapshot (#1146)

Sophie's Round-1 finding: the metadata pass iterated a snapshot taken before any write, so a survivor visited after the merge that rewrote it was normalized from its stale, un-merged copy and lost the merged data whenever it needed normalization itself. Every node and edge is re-read before it is decided, a merged survivor's user_id is normalized in the merge, and the regression arm runs the compatible merge in both visiting orders with a survivor that still carries the old user id and an old token."
- 2026-10-18T23:22:39Z @tobiu referenced in commit `c69476d` - "fix(migrations): rename tokens as whole handles, resolve chains, add the first-seat pairs (#1146) (#1152)

* fix(migrations): rename tokens as whole handles, resolve chains, add the first-seat pairs (#1146)

rewriteIdentityString ran two chained substring replaceAlls per pair, which was correct only while no old handle was a prefix of its target. The three first-seat pairs break that: @neo-fable became @neo-fable-mnemosyne-mnemosyne, @neo-gpt-emmy became @neo-gpt-euclid-euclid-emmy, and every re-run compounded. One alternation over all tokens, longest first, with the handle alphabet as the boundary, rewrites a whole handle once; a chain resolves to its last handle, so a second pass is a no-op. identityKey joins the metadata keys so an era-chain anchor would move with its root.

* fix(migrations): re-key compound identity row ids with their JSON ids and edge endpoints (#1146)

Euclid's read-only probe of the first head: a schema-valid IdentityState and EmbodiedEpisode pair kept its SQLite row ids while the JSON ids and the era anchor moved, and the HAS_EPISODE edge pointed at a row that no longer answered. Node and edge rows whose id embeds a handle now re-key together with their JSON id, edge endpoints follow, collisions are counted and skipped, and the era shape gets its own rule because the ISO since suffix hides the handle's right boundary behind a hyphen. The generic boundary stays strict; the protected @agent-neo-gpt rows are pinned unchanged.

* test(golden-path): the benched-owner arm seeds its own owner (#1146)

The arm expected OWNER_BENCHED_LANE for an issue assigned to a roster resident whose roots say operator_benched, but the detector reads participation from the worker's graph store, where sibling specs seed and clear AgentIdentity nodes; its verdict depended on which specs shared the worker, and this branch's schedule made it fail on three of four fresh CI runs while passing locally in every order. Seeding a dedicated benched owner inside the arm removes the dependency.

* fix(migrations): move compound rows under the Edges foreign keys and merge coexisting ids (#1146)

Sophie's probe with the production DDL: the Edges foreign keys reference Nodes(id) with ON DELETE CASCADE and no ON UPDATE, and the plane runs with foreign_keys on, so updating a connected compound row's primary key aborted the whole apply — the unit schema without the keys had let it pass. The re-key is now insert, move, delete, and when old and new compound ids coexist the old row merges into the surviving one (edges move, duplicates drop, the old row goes) instead of a skipped row whose edge JSON had already been rewritten. Edge JSON endpoints mirror the moved SQL columns. The spec schema carries the production keys and enforcement, a new arm covers the coexistence case, and every graph arm checks foreign_key_check empty.

* fix(migrations): merge coexisting compound rows with their provenance; a taken edge id keeps its own (#1146)

Sophie's provenance ask: a merge of coexisting compound rows is total and loses nothing — the surviving row's values win, the old row fills every gap, the older createdAt is kept; an edge whose re-keyed id is already taken keeps its own id and still takes its endpoints and metadata, and the collision is reported on every run. The structural edge pass now mirrors the JSON id onto the SQL id so the metadata pass re-keys both together or neither. Arms for both policies; the spec runs the production foreign keys.

* fix(migrations): a coexisting compound row merges only when compatible, otherwise fails closed (#1146)

Sophie's provenance finding, taken whole: a blind merge still discarded the old row's value wherever the two rows disagreed. The merge is now a policy — a key on one row is kept, a key on both must agree, the older createdAt is kept — and a disagreement fails closed for that row: nothing moves, nothing is written, the plan names the keys. Arms for the compatible merge and the fail-closed conflict.

* fix(migrations): a merged survivor is re-read, never overwritten from the pre-pass snapshot (#1146)

Sophie's Round-1 finding: the metadata pass iterated a snapshot taken before any write, so a survivor visited after the merge that rewrote it was normalized from its stale, un-merged copy and lost the merged data whenever it needed normalization itself. Every node and edge is re-read before it is decided, a merged survivor's user_id is normalized in the merge, and the regression arm runs the compatible merge in both visiting orders with a survivor that still carries the old user id and an old token."
- 2026-10-18T23:22:39Z @tobiu closed this issue
- 2026-10-18T23:24:58Z @neo-fable-mnemosyne referenced in commit `22c4cf0` - "feat(identity): the first seats' handles carry their Social Names; formerLogins keeps the trail (#1147)

Euclid, Julia and Mnemosyne were the first seat of their families and the only ones whose GitHub handle did not carry the Social Name every later seat was born with. The roots, the co-author map, the roster engine tags, the core-swarm user ids, the epoch-pinned migration maps, two lifecycle defaults, the ModelStats id rows and the README roster follow the login; a new optional formerLogins keeps every earlier handle so corpus author strings and old @mentions still resolve, and IssueIngestor counts them as internal authors. Data moves with renameAgentIdentities.mjs (#1146); this change deploys only inside the rename window, after the accounts are renamed."
- 2026-10-18T23:25:02Z @neo-fable-mnemosyne cross-referenced by #1147
- 2026-10-18T23:47:58Z @tobiu referenced in commit `b4b040d` - "feat(identity): the first seats' handles carry their Social Names; formerLogins keeps the trail (#1147) (#1155)

* feat(identity): the first seats' handles carry their Social Names; formerLogins keeps the trail (#1147)

Euclid, Julia and Mnemosyne were the first seat of their families and the only ones whose GitHub handle did not carry the Social Name every later seat was born with. The roots, the co-author map, the roster engine tags, the core-swarm user ids, the epoch-pinned migration maps, two lifecycle defaults, the ModelStats id rows and the README roster follow the login; a new optional formerLogins keeps every earlier handle so corpus author strings and old @mentions still resolve, and IssueIngestor counts them as internal authors. Data moves with renameAgentIdentities.mjs (#1146); this change deploys only inside the rename window, after the accounts are renamed.

* test(identity): roster-login fixtures follow the renamed handles (#1147)

Forty-six arms across twenty-one spec files pin neo-gpt, neo-gemini-pro or neo-fable as a registered roster login (trust tiers, benched-owner lanes, core-swarm summaries, wake routes, review-family resolution). They move to the renamed handles with the same whole-handle boundary rule the migration script uses, so sibling seats stay untouched.

* test(identity): realign two rewritten fixture blocks (#1147)"
- 2026-10-20T08:31:10Z @neo-opus-ada cross-referenced by #45

