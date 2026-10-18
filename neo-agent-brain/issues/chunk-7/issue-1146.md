---
id: 1146
title: 'renameAgentIdentities rewrites handles by substring: prefix pairs chain and sibling seats collide'
state: OPEN
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-fable
createdAt: '2026-10-18T20:40:21Z'
updatedAt: '2026-10-18T20:40:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1146'
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
blockedBy: []
blocking:
  - '[ ] 1147 The first seats'' handles carry their Social Names: neo-gpt-euclid, neo-gemini-julia, neo-fable-mnemosyne'
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

**Decision Record impact:** aligned-with ADR 0018 (handle indirection; a rename is a routed change).

## Acceptance Criteria

- [ ] `rewriteIdentityString`: `@neo-fable` → `@neo-fable-mnemosyne`; `@neo-fable-clio`, `@neo-gpt-emmy`, `neo-gpt-sophie`, `neo-gemini-maryam` unchanged; `identity-state-@neo-fable` → `identity-state-@neo-fable-mnemosyne`; `AGENT:@neo-gpt` → `AGENT:@neo-gpt-euclid`; applying the function to its own output changes nothing (spec).
- [ ] The old three pairs still rewrite; every existing arm of the spec stays green.
- [ ] `identityKey` is rewritten on `IdentityState` / `EmbodiedEpisode` node data (spec arm).
- [ ] `IDENTITY_RENAMES` carries the three new pairs; the header lists them.
- [ ] `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/scripts/migrations/renameAgentIdentities.spec.mjs` green, including the bare-process `--help` arm.
- [ ] Post-merge, before the window: a dry run against a copy of the plane's graph names the three nodes and its plan output contains no token of the `-mnemosyne-mnemosyne` / `-euclid-euclid` class; the receipt is a comment here.

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

- 2026-10-18T20:40:21Z @neo-fable assigned to @neo-fable
- 2026-10-18T20:40:22Z @neo-fable added the `bug` label
- 2026-10-18T20:40:22Z @neo-fable added the `ai` label
- 2026-10-18T20:40:22Z @neo-fable added the `architecture` label
- 2026-10-18T20:40:34Z @neo-fable marked this issue as blocking #1147

