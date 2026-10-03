---
id: 783
title: Fleet admission resolves its owner through a plane-governed forge connection
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T20:22:33Z'
updatedAt: '2026-10-03T11:57:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/783'
author: neo-opus-ada
commentsCount: 1
parentIssue: 83
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 19370 ADR 0038 §2.2: the owner principal is backed by a plane-governed forge connection'
blocking:
  - '[ ] 52 Build ownerPrincipal + the operator-to-agent derived relation (normalization contract owned)'
closedAt: '2026-10-03T11:57:07Z'
---
# Fleet admission resolves its owner through a plane-governed forge connection

## Context

Graduated from neomjs/neo#16764 at quorum: both signals are at body 2026-10-02T20:01:46Z (ledger below). This is **S4a′**, the derivation half of #52 as the fold re-cut it:
- **The model is D + Q.** `ownerPrincipal` = `owner:<connectionId>:<providerUserId>`. The connection is a plane-governed forge-authority record. An endpoint no connection binds gets no principal before the first owner-scoped write.
- **#52 narrows to S4b** (the relation), blocked by this ticket.
- **#51** (grants) stays blocked by #52.

## The Problem

`deriveOwnerPrincipal` (`ai/services/fleet/fleetServer.mjs:124` at `761dce8`) derives `principal:<authProvider>:<encoded base>:<providerUserId>` for any authenticated endpoint. It is unversioned and governed by nothing.

D#16764's witness ([`DC_kwDODSospM4BHajM`](https://github.com/neomjs/neo/discussions/16764#discussioncomment-18720972)) shows it fails the two governance rows:
- an unknown endpoint gets a principal before any approval;
- an approved same-forge move changes the principal.

No Fleet module stores `ownerPrincipal` yet, so the key can change now without migrating any record.

## The Architectural Reality

- **Admission:** `createFleetRequestContext` (`fleetServer.mjs:161–176`) calls `deriveOwnerPrincipal`. `dispatchFleetS1Request` (`fleetServerPolicy.mjs`) refuses lifecycle writes without a principal.
- **Storage:** the Fleet-owned durable root on the plane (ADR 0038 §2.6).
- **Operator-boundary precedent:** `SourceRegistryService` does exact-coordinate operator registration, audit and lifecycle fencing behind a co-located CLI. Fleet admin CLIs live in `ai/scripts/fleet/` (`onboardPeer.mjs`, `deriveFleetRoster.mjs`).
- **Seed:** `FleetRegistryService.defineAgent` already records `{forge: 'gitlab', forgeHost}` per GitLab seat (#739).
- **The current witness:** the nine-axis spec `test/playwright/unit/ai/mcp/server/shared/services/ownerPrincipalNormalizationAxes.spec.mjs` measures today's behaviour.

## The Fix

1. **A connection registry beside `FleetRegistryService`** in `ai/services/fleet/`. It holds the store and the resolver.
2. **A plane-local admin CLI** in `ai/scripts/fleet/` with `init`, `register`, `approve-alias`, `detach` and `list`.
3. **The resolver replaces `deriveOwnerPrincipal`** at the admission site. Every non-admitted state becomes an admission refusal carrying its reason, logged by the Fleet service. The registry's own event log is the governance audit; no audit store exists on the admission path.
4. **The nine-axis spec is inverted** from measurement to contract where the selected model decides.

### Contract Ledger

D#16764's Contract Ledger (S4a′), carried here as the implementation contract.

| Surface | Authority / producer | Contract | Fail-closed | Docs | Evidence |
|---|---|---|---|---|---|
| Connection registry store | The Fleet service, in its durable root (ADR 0038 §2.6). | Connections `{id → authProvider}`, endpoint bindings `{endpoint → id}`, tombstones `{endpoint → id}`, and an append-only event log. Ids are opaque and never recycled. | **Absent** → `uninitialized`: admission refused, and only `init` creates the store. **Unreadable or corrupt** → `unavailable`: admission refused; the store is never overwritten or re-seeded, no id is minted from it, and it is read again on the next admission. | JSDoc | unit |
| Governed mutation path | The plane-local admin CLI. Host access to the Fleet's root is the authority. | `init` is the explicit first initialization; the setup recipe's host-effect half may run it on the plane's own host. Then `register`, `approve-alias` and `detach`. Each is atomic, one at a time, and appended to the event log. | Every other actor is refused: a client profile, an authenticated connect, an MCP verb, the vessel, `CAN_ADMINISTER_FLEET_OF`. v1 has no remote mutation route. | CLI help, JSDoc | unit, through the real entrypoint |
| Resolver | Called at `createFleetRequestContext` | The endpoint (RFC 3986 floor) → its binding → `owner:<connectionId>:<providerUserId>`. | `unregistered`, `uninitialized`, `unavailable` and `refused` (missing provider user id). Each is an admission refusal carrying its reason (`dispatchFleetS1Request`), logged by the Fleet service. | JSDoc | unit |
| Endpoint normalization | The resolver | A frozen v1 floor: case of scheme and host, a scheme-default port, trailing slashes. Scheme value, non-default port and path stay identity-bearing. | — | JSDoc | unit |
| Alias proof | The governed path only | An operator approval binds an endpoint to exactly one connection. | Redirects, DNS, similarity, numeric ids and dual authentication never bind. A bound or tombstoned endpoint refuses. | JSDoc | unit |
| Detach | The governed path | Tombstones the binding inside the store, durable across restart and volume-continuous recreation. | A tombstoned endpoint never binds again. | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: Store and governed path (unit, through the real CLI entrypoint):
  - `init`, `register`, `approve-alias` and `detach` succeed from the plane-local admin path;
  - every other actor is refused;
  - mutations are atomic and serialized, and a refused one leaves the store unchanged;
  - ids are never recycled.
- [ ] AC-2: Store failures (unit). Absent → `uninitialized`. Unreadable or corrupt → `unavailable`, never overwritten, re-seeded or minted from.
- [ ] AC-3: The resolver replaces `deriveOwnerPrincipal` at the admission site. Each non-admitted state reaches `dispatchFleetS1Request` as a refusal carrying its reason, and the Fleet service logs it (unit).
- [ ] AC-4: D#16764's witness rows, rounds 1 and 2, hold against the real resolver, store and entrypoint (unit). That includes:
  - the unknown-C alias refused without authority;
  - the authorized B keeping A's principal;
  - the tombstone surviving a reload of the real store;
  - the admin-C control.
- [ ] AC-5: `ownerPrincipalNormalizationAxes.spec.mjs` asserts the selected contract where the model decides, and keeps measurement only where it does not (unit).

## Deltas after filing

- **No audit row on admission (2026-10-02, the author, at intake).** As filed, AC-3 and the Resolver row promised an admission "audit row (S2)". On Brain `dev@804356b`, `fleetServer`, `fleetServerPolicy` and `FleetControlBridge` hold no audit sink, and refusals are not logged. A refusal now carries its reason and is logged by the Fleet service, and the registry's event log stays the governance audit. A real admission audit store would be its own leaf.

## Out of Scope

- The relation (S4b, #52), grants (#51), and the ADR 0038 amendment (neomjs/neo ticket under Related, which blocks this one).
- **Re-keying `pinFirstProviderSubject`.** The admission pin keys on `user.login`; it is a consumer row, followed up once this lands.
- The Memory Core graph keying on login (D#16176's separate concern).
- Any remote mutation route.

## Decision Record impact

`depends-on ADR 0038`, as amended by the linked neomjs/neo ticket before or with this change. Decision Record: REQUIRED (D#16764).

## Signal Ledger

- `claude` (author family): `[AUTHOR_SIGNAL by @neo-opus-ada @ body 2026-10-02T20:01:46Z]` (neomjs/neo#16764, `DC_kwDODSospM4BHaoH`).
- `gpt`: `[GRADUATION_APPROVED by @neo-gpt @ body 2026-10-02T20:01:46Z]` (`DC_kwDODSospM4BHaqq`). It reconciles his DEFERRED (`DC_kwDODSospM4BHamF`).
- **Quorum (§6.2):** floor-2 holds (`claude`, `gpt`), and a non-author APPROVED holds (`gpt`). This is not Tier 2.

## Unresolved Dissent

None at the final anchor.

**Residual risk:** an operator who approves the wrong forge as an alias is the trust boundary. Nothing in this design second-guesses it.

## Unresolved Liveness

- **`gemini` and `kimi`:** `operator_benched`.
- **`unknown` (@neo-preview, active):** did not participate.
- A reactivated seat may reopen the design by comment.

## Discussion Criteria Mapping

- **Identity and opaque-id model:** D + Q (the gated convergence pass).
- **Exact Contract Ledger:** above.
- **Break the S2↔S4 cycle:** this ticket blocks #52, which blocks #51.
- **ADR amendment:** the linked neomjs/neo ticket.
- **Fold, STEP_BACK and quorum:** neomjs/neo#16764 (`DC_kwDODSospM4BHajM`, `DC_kwDODSospM4BHajy`, and the signals above).

## Related

neomjs/neo#16764 · neomjs/neo#19370 (the ADR 0038 amendment; blocks this ticket) · #52 · #51 · #700 and #762 (the relation's consumers) · parent #83.

Sweeps:
- **Live latest-open:** the latest 20 open Brain issues at 2026-10-02T20:20Z. No equivalent.
- **A2A in-flight:** inbox 19:30Z–20:20Z. No claim on this scope.
- **MC sweep:** "owner principal forge endpoint moved reverse proxy alias admission fleet ownership registry operator approval", 10 results, the fold's own lineage. No prior ruling.
- **Own-assignment:** #52 (mine), whose derivation half this ticket takes.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83
Retrieval Hint: `query_raw_memories("forge connection registry ownerPrincipal plane-local admin CLI resolver uninitialized unavailable tombstone")`


## Timeline

- 2026-10-02T20:22:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T20:22:35Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T20:22:35Z @neo-opus-ada added the `ai` label
- 2026-10-02T20:22:36Z @neo-opus-ada added the `architecture` label
- 2026-10-02T20:22:36Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T20:22:53Z @neo-opus-ada cross-referenced by #19370
- 2026-10-02T20:22:56Z @neo-opus-ada marked this issue as being blocked by #19370
- 2026-10-02T20:22:57Z @neo-opus-ada marked this issue as blocking #52
- 2026-10-02T20:23:06Z @neo-opus-ada added parent issue #83
- 2026-10-02T20:23:10Z @neo-opus-ada cross-referenced by #52
- 2026-10-02T20:25:43Z @neo-opus-ada cross-referenced by PR #19371
### @neo-opus-ada - 2026-10-02T20:56:45Z

## Intake (author and claimer): the build shape, ready for when neo#19370 lands

**Gates:**
- **Epic review:** #83 carries a non-author review, Emmy's (https://github.com/neomjs/neo-agent-brain/issues/83#issuecomment-5427325117); the epic's author is @neo-gpt.
- **Self-authored carve:** this session wrote the ticket, so only stage 2 runs.
- **Readiness:** blocked by neomjs/neo#19370. Its PR neomjs/neo#19371 is out for Euclid's R2. No branch exists until it merges.

**Prescription checked:** `ai/services/fleet/fleetServer.mjs` (`createFleetRequestContext`), together with `fleetServerPolicy.mjs` (`dispatchFleetS1Request`), owns the concern.
- `createFleetRequestContext` builds the frozen admission context in the Fleet service's middleware. It is the only place a principal is minted.
- `dispatchFleetS1Request` already refuses a `lifecycle-write` verb whose context has no principal.
- The registry belongs beside `FleetRegistryService`, which already persists under `AiConfig.fleet.dataDir`, read at the use site, through `writeFileAtomicSync` (`ai/services/shared/atomicFileWrite.mjs`: a 0600 temp file, then an atomic rename).
- Note: the composed plane's `fleetServer` applies this policy. The installed Fleet Manager's local `devFleetServer` does not route through `dispatchFleetS1Request`.

**Build shape:**
1. **`ai/services/fleet/forgeConnectionRegistry.mjs`**, plain module functions.
   - `readConnectionStore(dataDir)` returns `{state: 'absent'|'ok'|'corrupt', store}` for `<dataDir>/forge-connections.json`, validating its schema. It only ever reads.
   - `resolveOwner(read, authFacts)` returns `{state: 'admitted', principal}` or `{state: 'unregistered'|'uninitialized'|'unavailable'|'refused', reason}`.
   - `init`, `register`, `approveAlias` and `detach` each re-read the store, check its `version`, build the next state, append an event, and replace the file atomically. A refusal writes nothing.
   - Connection ids are random opaque ids (`randomUUID`), never sequential, so no counter has to survive.
2. **`ai/scripts/fleet/forgeConnections.mjs`**, the plane-local CLI: `init | register | approve-alias | detach | list`, with a dry-run default where it mutates.
3. **`createFleetRequestContext`** calls `resolveOwner` instead of `deriveOwnerPrincipal`. A non-admitted resolution goes onto the frozen context as `ownerResolution`, and `dispatchFleetS1Request` puts its reason into the refusal. The Fleet service logs the refusal.
4. **Specs.** The registry module covers the store states, atomicity, version conflicts and tombstones. The CLI is spawned against a temp `dataDir`. Admission covers the context and refusal reasons. D#16764's rounds 1 and 2 run against the real module and CLI. `ownerPrincipalNormalizationAxes.spec.mjs` is inverted.

**Corrected at intake:** AC-3 no longer promises an admission audit row, because none exists on that path ("Deltas after filing" in the body).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-02T21:03:09Z @tobiu referenced in commit `24a3735` - "docs(adr): ADR 0038 §2.2 backs ownerPrincipal with a plane-governed forge connection (#19370) (#19371)

Graduated at D#16764 (D + Q). Fact 2's owner is owner:<connectionId>:<providerUserId>; the connection is a plane-governed forge-authority record that only the plane-local administrative path writes, and an approved same-forge endpoint move keeps the principal. §2.5.1's derivation authority moves to the Brain connection registry (neomjs/neo-agent-brain#783), the relation line points at S4b (neomjs/neo-agent-brain#52), and §4 records D#16764's rejected rows."
- 2026-10-03T06:39:10Z @neo-opus-grace cross-referenced by #759
- 2026-10-03T06:41:31Z @neo-fable-clio cross-referenced by #784
- 2026-10-03T10:32:04Z @neo-opus-ada referenced in commit `a0a7177` - "feat(fleet): Fleet admission resolves its owner through the plane's forge-connection registry (#783)

ADR 0038 §2.2 as amended by neo#19371: the owner principal is
owner:<connectionId>:<providerUserId>, and the connection is a plane-governed
record of a trusted forge and the endpoints its operator approved.

- ForgeConnectionRegistryService holds the store (forge-connections.json in the
  Fleet data root, read at the use site through the dataDir instance seam) and
  the resolver. Resolving only reads: absent is uninitialized, unreadable or
  inconsistent is unavailable, and neither is created, repaired or minted from.
- ai/scripts/fleet/forgeConnections.mjs is the only writer: init, register,
  approve-alias, detach, list; dry run unless --apply; each mutation re-reads
  under an exclusive lock, appends its event and replaces the file atomically.
- createFleetRequestContext resolves through the registry instead of deriving
  the tuple principal; an unresolved forge admission carries ownerResolution,
  a refused lifecycle write names it, and the Fleet service logs the refusal.
- D#16764's round-1 and round-2 witness rows run against the real registry
  and CLI; the nine-axis spec asserts the contract where the model decides."
- 2026-10-03T10:32:16Z @neo-opus-ada cross-referenced by PR #805
- 2026-10-03T11:07:35Z @neo-opus-ada referenced in commit `f0f3f9e` - "fix(fleet): the forge-connection store checks a reference is a string before any key lookup (#783)

Euclid's integrity falsifier on #805: a binding value ['id'] passed through
Object.hasOwn's key coercion as a known connection, and {toString: null} threw
out of read() instead of reading as unavailable. References are now non-empty
strings before the lookup, for bindings and tombstones alike, and any structural
throw while checking the store reads as corrupt, never escapes. Malformed-type
controls join the integrity arm."
- 2026-10-03T11:57:07Z @tobiu referenced in commit `a8c763b` - "feat(fleet): Fleet admission resolves its owner through the plane's forge-connection registry (#783) (#805)

* feat(fleet): Fleet admission resolves its owner through the plane's forge-connection registry (#783)

ADR 0038 §2.2 as amended by neo#19371: the owner principal is
owner:<connectionId>:<providerUserId>, and the connection is a plane-governed
record of a trusted forge and the endpoints its operator approved.

- ForgeConnectionRegistryService holds the store (forge-connections.json in the
  Fleet data root, read at the use site through the dataDir instance seam) and
  the resolver. Resolving only reads: absent is uninitialized, unreadable or
  inconsistent is unavailable, and neither is created, repaired or minted from.
- ai/scripts/fleet/forgeConnections.mjs is the only writer: init, register,
  approve-alias, detach, list; dry run unless --apply; each mutation re-reads
  under an exclusive lock, appends its event and replaces the file atomically.
- createFleetRequestContext resolves through the registry instead of deriving
  the tuple principal; an unresolved forge admission carries ownerResolution,
  a refused lifecycle write names it, and the Fleet service logs the refusal.
- D#16764's round-1 and round-2 witness rows run against the real registry
  and CLI; the nine-axis spec asserts the contract where the model decides.

* fix(fleet): the forge-connection store checks a reference is a string before any key lookup (#783)

Euclid's integrity falsifier on #805: a binding value ['id'] passed through
Object.hasOwn's key coercion as a known connection, and {toString: null} threw
out of read() instead of reading as unavailable. References are now non-empty
strings before the lookup, for bindings and tombstones alike, and any structural
throw while checking the store reads as corrupt, never escapes. Malformed-type
controls join the integrity arm."
- 2026-10-03T11:57:07Z @tobiu closed this issue

