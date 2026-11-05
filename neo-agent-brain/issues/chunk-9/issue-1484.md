---
id: 1484
title: A plane's boot seed holds no team identity
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T14:03:29Z'
updatedAt: '2026-11-05T18:15:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1484'
author: neo-opus-vega
commentsCount: 1
parentIssue: 875
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-05T18:15:55Z'
---
# A plane's boot seed holds no team identity

## Context

This is class A of #875's census: who exists. I took the census on 2026-10-07 and re-checked it on 2026-11-05 at Brain `dev` `ba1cb87b`. The operator's ruling behind #875 is that runtime identity facts come from the plane's graph.

Class A is the first fact an outside operator meets: their plane boots holding our team's identities. On 2026-11-05 that was 14 AgentIdentity roots, `@tobiu` at `owner` among them. This lands on the Fleet Manager v1 outside-operator journey.

**Re-scoped 2026-11-05.** A PR resolves exactly one ticket. So this leaf keeps the boot seed, and the four candidate readers moved to #1485. The intake record is comment 6663307247.

## The Problem

`GraphService.provisionMissingBootSeeds()` provisions `createGraphBootSeedManifest()`. That manifest's nodes are the two anchors plus every `identityRoots.IDENTITIES` entry (`ai/graph/bootSeedManifest.mjs:54`). Every plane therefore boots with our roster.

## The Architectural Reality

- Design authority: #875's recorded ruling (the operator, 2026-10-05, on #874), *"agent os and MCP tools should never use them. this only works for OUR own team, but can not work for other operators who simply do not even have that file for them."*
- One manifest serves two callers: boot, and the restore-empty-target freshness proof (`restoreTargetSetStorage.mjs` `inspectGraphBootFreshness`).
- Two roster entries are not team members: `@system`, the lifecycle sender, and `AGENT:*`, the broadcast sentinel whose node the mailbox's edges need (#10174, #11029). Every plane needs both.
- Team identities already have two ways onto a plane:
  - A principal's node is auto-provisioned at its first authenticated request under `auth.autoProvisionIdentitySources` (`Server.ensureAgentIdentityForAuthContext`).
  - Our team's plane is seeded by `ai/scripts/setup/seedAgentIdentities.mjs`, which `onboardPeer` already names.
- `GraphService.provisionMissingIdentityRoots()` has no runtime caller; the manifest replaced it.
- Several roster readers build trust maps that include `@system`, so the roster keeps listing the two roots. Removing them is class B work.
- `generateRosterOnboarding` inserts a new resident before the `AGENT:*` entry, and its spec loads the applied roster as a `data:` URL.
- The trust taxonomy lives in `identityRoots.mjs`, so a module declaring the two roots once cannot import it without closing a cycle.

## The Fix

1. Move `TRUST_TIERS` and `TRUST_TIER_ORDER` to `ai/graph/trustTiers.mjs`; the roster re-exports them for their existing importers.
2. Declare the two roots once, in `ai/graph/mailboxRoots.mjs` (`SYSTEM_SENDER`, `BROADCAST_SENTINEL`):
   - the manifest's fixed nodes take them, the roster stops being a manifest input, and the manifest version becomes 2;
   - the roster lists the same two objects first and last;
   - the generator anchors on the `BROADCAST_SENTINEL` line.
3. Delete `provisionMissingIdentityRoots`, and with it GraphService's roster import.
4. Correct `OwnAgentTeam.md` and the header of `identityRoots.mjs`, which still say boot seeds the roster.

## Contract Ledger

Added 2026-11-05 at review RA-2 (5789624471). No runtime mechanism is new; every row names the surface #1486 changes or keeps.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Boot seed manifest: `createGraphBootSeedManifest()`, `GRAPH_BOOT_SEED_VERSION = 2` (`ai/graph/bootSeedManifest.mjs`) | #875's ruling ([5992651676](https://github.com/neomjs/neo-agent-brain/issues/874#issuecomment-5992651676)) | Four fixed nodes (`frontier`, `Neo-Master-Architecture`, `@system`, `AGENT:*`) and one edge (`frontier` → `Neo-Master-Architecture`, `SYSTEM_TENET`); no AgentIdentity node; the roster is no input. | None: the manifest is fixed. | module JSDoc | `bootSeedManifest.spec.mjs`; `GraphService.spec.mjs` "boot provisions the mailbox roots and no team identity via initAsync" (AC-1) |
| Mailbox roots: `SYSTEM_SENDER`, `BROADCAST_SENTINEL` (`ai/graph/mailboxRoots.mjs`) | this ticket, Fix 2 | One declaration, imported by the manifest and by the roster (listed first and last). | None. | module JSDoc | AC-2 spec arm; the generator anchors on `BROADCAST_SENTINEL` (`generateRosterOnboarding.spec.mjs`) |
| Additive boot: `GraphService.provisionMissingBootSeeds(manifest)` | this ticket, Fix 3 | Creates the manifest's missing nodes and edges only. It never rewrites or deletes an existing node or edge, so a populated graph keeps every identity it holds. | Throws when the graph storage database is missing. | method JSDoc | `GraphService.spec.mjs` "boot provisioning adds missing seeds without rewriting a persisted one"; AC-4 on our plane (post-merge) |
| Restore-target freshness: `evaluateGraphBootSeedFreshness()` | the manifest's fingerprint contract | A target is fresh only when its persisted nodes and edges fingerprint-equal the v2 manifest. | A target initialized under v1 holds the roster (18 nodes / 1 edge) and is refused: "persisted graph differs from boot seed v2 (nodes 18/4, edges 1/1)". Booting it again cannot help (row 3), so a restore needs a new empty target initialized under v2. A populated graph is never cleared to pass the proof. | function JSDoc; #1486 body | the review's isolated probe (5789624471); `bootSeedManifest.spec.mjs` |
| Trust tiers: `TRUST_TIERS`, `TRUST_TIER_ORDER` (`ai/graph/trustTiers.mjs`) | this ticket, Fix 1 | Declared once; `identityRoots.mjs` re-exports both, so every existing import resolves unchanged. | None. | module JSDoc | CI: the existing importers' specs |
| Team seeding: `node ai/scripts/setup/seedAgentIdentities.mjs` | #875's ruling; this ticket, Fix 4 | The only path by which a roster reaches a fresh Memory Core, and how it is refreshed after a roster change. Boot never seeds it. | A team without a roster skips the step. | `learn/agentos/OwnAgentTeam.md` (corrected in #1486) | the guide's diff; `generateRosterOnboarding.spec.mjs` |

## Acceptance Criteria

- [ ] AC-1: A unit arm shows a fresh plane's boot provisions the two anchors and the two mailbox roots, and no AgentIdentity node.
- [ ] AC-2: The manifest and the roster take the two mailbox roots from the one declaration in `mailboxRoots.mjs`.
- [ ] AC-3: Neither `bootSeedManifest.mjs` nor `GraphService.mjs` imports `identityRoots`. The runtime modules that import it, outside `ai/scripts/`, drop from 18 to 16. The earlier "24" counted files that mention it.
- [ ] AC-4 (post-merge): on our own plane, the existing identity nodes are unchanged after a Memory Core restart on the new seed.

## Out of Scope

- The heartbeat, issue-focus, maintainer and open-work readers, which moved to #1485.
- Classes B (trust tier), C (family: #700), D (display) and E (vocabulary, team tooling).
- Removing the mailbox roots from the roster, which is class B: its readers' trust maps include `@system`.
- Deleting existing nodes.
- The Fleet admission writer (#1297).

## Decision Record impact

`aligned-with` #875's ruling. No ADR changes.

## Related

#875 (parent) · #1485 (the readers) · #700 / #1296 / #1297 (class C and admitted membership) · #663 (precedent: a write guard that stopped requiring a roster entry) · #1161 (boot seeds in specs)

Live latest-open + closed sweep: the latest 20 open Brain issues, plus searches ("boot seed roster identities", "heartbeat candidates identity nodes", "issue focus roster", "who exists identityRoots", `provisionMissingBootSeeds`), at 2026-11-05T14:0xZ. No equivalent; Out of Scope read on #663. No A2A claim on identity in the last hour. MC: the class C consumer belongs to #700 (Grace 10-01, Sophie's chain), not this leaf. Own open assignments: #875 is the parent, and nothing else covers this surface.

Origin Session ID: 57180f73-f0b4-4ae5-bac8-095fd4813333

Retrieval Hint: "who exists identity nodes not static roster boot seed mailbox roots class A"

Authored by Vega (Opus 5.5, Claude Code)



## Timeline

- 2026-11-05T14:03:29Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T14:03:30Z @neo-opus-vega added the `enhancement` label
- 2026-11-05T14:03:30Z @neo-opus-vega added the `ai` label
- 2026-11-05T14:03:30Z @neo-opus-vega added the `architecture` label
- 2026-11-05T14:03:31Z @neo-opus-vega added the `agent-os` label
- 2026-11-05T14:03:43Z @neo-opus-vega cross-referenced by #875
### @neo-opus-vega - 2026-11-05T16:28:15Z

### Intake: drift probe and Stage 2

**Drift probe.** `git log origin/dev --since=2026-11-05T14:03Z` at Brain `40a2f62c` shows 2 commits (`deferencePhraseMatch`, a deps bump). Neither touches this ticket's paths, so nothing moved underneath it.

**Stage 2, the prescription, module by module:**

- **`ai/graph/bootSeedManifest.mjs` owns the concern.** It is the one manifest behind both boot and the restore-empty-target freshness proof. Fix 1 needs two corrections.
  - The roster also carries the two roots that are not team members: `@system` (the lifecycle sender) and `AGENT:*` (the broadcast sentinel, #10174 / #11029). Every plane needs both, so they stay in the seed.
  - "Its operator identity" has no runtime source. No config names a plane's operator. The operator's node arrives like every principal's: auto-provisioned at the first PAT sign-in (`Server.ensureAgentIdentityForAuthContext`). Its trust tier is class B.
  - So the seed becomes the two anchors plus the two system roots, with no AgentIdentity. AC-1 reads that way.
- **`swarmHeartbeat.mjs` owns its concern, and the problem is narrower than the table says.** The default source, `active-a2a-participants`, is already graph-derived. Only the opt-in `active-local-team` reads the roster, and the participation read it needs is already supplied.
- **`issueFocusSections.getStaleAssignmentMaintainers()` owns its concern.** The nodes are already read at `:1162`.
- **`IssueIngestor.internalAuthorLogins` owns its concern.** It is built from the roster at import time.
- **`wireFleetOpenWorkSource.seatIdentities` owns its concern.** `byLogin` already prefers the registry. `byName`, which resolves the PR body's author line, resolves through the roots only.
- **Deletion:** `GraphService.provisionMissingIdentityRoots` has no runtime caller, because the manifest replaced it. It is the only reason GraphService imports the roster.

**Beneficiary.**
- An outside operator has nothing new to do. Their plane stops holding our 14 AgentIdentity roots, `@tobiu` at `owner` among them.
- Our team has one new step, on a fresh plane only. The roster is seeded by `node ai/scripts/setup/seedAgentIdentities.mjs`, the path `onboardPeer` already names, instead of by boot.
- That step follows from the 10-05 ruling that no runtime reader may use the roots.

**Plan (re-scoped, since a PR resolves exactly one ticket).**
- This ticket keeps Fix 1, the boot seed. Its body now carries that fix's ACs.
- Fixes 2 and 3, the four readers, moved to #1485.
- `@system` and `AGENT:*` are declared once, in `ai/graph/mailboxRoots.mjs`. Both the manifest and the roster take them from there.
- The roster still lists both roots, because several of its readers build trust maps that include `@system`.
- The trust taxonomy moves to `ai/graph/trustTiers.mjs`. Without that move, the new module and the roster would import each other.
- The roster generator now anchors on the `BROADCAST_SENTINEL` line.

— Vega (Opus 5.5, Claude Code) 🌿


- 2026-11-05T16:39:11Z @neo-opus-vega cross-referenced by #1485
- 2026-11-05T16:39:24Z @neo-opus-vega changed title from **The plane's identity nodes, not the static roster, decide who exists** to **A plane's boot seed holds no team identity**
- 2026-11-05T16:49:08Z @neo-opus-vega cross-referenced by PR #1486
- 2026-11-05T17:16:42Z @neo-opus-vega referenced in commit `493d366` - "test(lifecycle): the cooldown spec's subprocesses read a graph that holds their coordinator (#1484)

The spec addressed @neo-opus-ada because boot used to seed the roster into
the subprocess graph. Boot seeds no team identity now, so each test
writes a scenario graph holding @test-coordinator and hands it to the
subprocesses through NEO_MEMORY_DB_PATH_TEST, as checkSunsetted.spec does."
- 2026-11-05T18:03:09Z @neo-gemini-julia cross-referenced by PR #1487
- 2026-11-05T18:15:55Z @tobiu referenced in commit `5b04243` - "fix(graph): a plane's boot seed holds no team identity (#1484) (#1486)

* fix(graph): a plane's boot seed holds no team identity (#1484)

Every plane booted holding our roster: createGraphBootSeedManifest() took
identityRoots.IDENTITIES as its node input. So an outside operator's
Memory Core provisioned our 14 AgentIdentity roots, @tobiu at owner
among them.

The fixed nodes are now the two anchors plus the system sender and the
broadcast sentinel, the two roots every plane's mailbox needs. Both are
declared once, in ai/graph/mailboxRoots.mjs. The manifest and the roster
take the same two objects from there. The roster keeps listing them
because its readers' trust maps include @system. The manifest version
becomes 2.

The trust taxonomy moves to ai/graph/trustTiers.mjs so the new module
needs no import from the roster; the roster re-exports both constants
for their importers. The roster generator inserts a new resident before
the BROADCAST_SENTINEL line, and its spec resolves the applied roster's
sibling imports before the data: URL import.

GraphService.provisionMissingIdentityRoots() had no runtime caller once
the manifest took over boot. It goes, and with it GraphService's roster
import. Its byte-stability falsifier now runs against
provisionMissingBootSeeds().

* test(lifecycle): the cooldown spec's subprocesses read a graph that holds their coordinator (#1484)

The spec addressed @neo-opus-ada because boot used to seed the roster into
the subprocess graph. Boot seeds no team identity now, so each test
writes a scenario graph holding @test-coordinator and hands it to the
subprocesses through NEO_MEMORY_DB_PATH_TEST, as checkSunsetted.spec does."
- 2026-11-05T18:15:56Z @tobiu closed this issue

