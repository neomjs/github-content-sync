---
id: 699
title: Retire the Fleet's local stdio Memory Core and Knowledge Base target
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - architecture
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T15:23:13Z'
updatedAt: '2026-10-03T11:23:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/699'
author: neo-opus-vega
commentsCount: 1
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T21:06:59Z'
---
# Retire the Fleet's local stdio Memory Core and Knowledge Base target

## Context

Operator, 2026-10-01 ~15:15Z, in Vega's session: *"stdio MC MCP server might be a relict (no longer supported) from before agent os dockerization. so, if this is still a setting inside FM, i would challenge if it should get removed there."* Then: *"(same for KB)"*. The history behind it, from the operator: stdio servers came first. The dockerized plane was built for a cloud deployment, and the local Agent OS later moved onto the same containers, before the repo split.

It is still a setting, and it is the Fleet default (Brain dev `ab0846e`, Institution dev `27a44a0`):
- `FleetRegistryService.defineAgent` stores `mcpTarget: null` ("resident") unless a tenant is passed (`ai/services/fleet/FleetRegistryService.mjs:326`).
- For that target the seat plan materializes `memory-core` and `knowledge-base` as local stdio servers: `transport: resource ? 'streamable-http' : 'stdio'` (`ai/services/fleet/managedAgentWorkspacePlan.mjs:175`).
- Institution's agent config offers "local" beside connected tenants (`apps/agentos/view/fleet/detail/AgentConfigComponent.mjs:197-200`).

Adjacent precedent: neo `#15805` (closed 2026-07-27) added the remote-HTTP opt-in and kept stdio as the default. It explicitly left "FM-managed fleet generation" out of scope, so the Fleet's default was never decided there.

## The Problem

A resident row runs a per-seat stdio Memory Core and Knowledge Base, which resolve their store from local placement rather than from the plane the Fleet serves. The consequence depends on the FM's plan (`resolveProductBrainPlan`, Institution `harness/brain.mjs:171`):
- **`plane-attach`** (`fleet.planeBase` set, e.g. the local dockerized Agent OS). The plane's store sits in a Docker volume that a host process cannot open (see the doc comments of `ai/daemons/wake/readSubscriptionsOverMcp.mjs` and `armSeatWakeRoute.mjs`). A resident seat therefore writes memories no peer reads, reads a mailbox no peer writes to, and gets no plane-dispatched wakes (#79).
- **Unresolved placement.** #669: a resident Claude Desktop seat wrote into the app bundle, and recovery needed a WAL replay into the plane (PR #692, AC-6).
- **`own`** (the FM owns a per-user plane, Institution `#347`, ADR 0019 §10.7). This is a supported plane, not a private fork. Its seats still reach it through per-seat stdio processes rather than through the plane's served endpoints.

## The Architectural Reality

- Only `memory-core` and `knowledge-base` are plane resources (`MCP_TARGET_RESOURCE_KEYS`, `managedAgentWorkspacePlan.mjs:123`). `github-workflow`, `gitlab-workflow` and `neural-link` stay local stdio servers, and their placement is #692's.
- **Credentials differ by plane.** The local dockerized plane validates GitHub PATs (`deploy/cloud/docker-compose.local-agent-os.yml:61`). A tenant stores a distinct plane bearer and proves its subject before spawn (`startAgentProvisioned.mjs:193-216`). No plane may be assumed to accept a forge PAT.
- Six of the seven launchable harness types declare `tenantMcpTarget: true`; Antigravity declares `false` (`src/fleet/contract/harnessTypes.mjs:14-20`).
- A tenant target requires a managed repo (`startAgentProvisioned.mjs:144`).

## The Fix

1. A seat's MC and KB are always rows on the plane the Fleet serves: the attached plane in `plane-attach` and `attach`, and the own plane's served endpoints in `own`. The Fleet stops materializing per-seat stdio MC/KB servers. If `own` mode serves no HTTP endpoint today, the lane either adds one or keeps `own` as the one named exception with its reason. That decision is made in the lane and stated on the PR.
2. Each seat authenticates with the credential class its plane accepts, proven to resolve to the seat before spawn (the tenant readiness gate). A forge PAT is never assumed.
3. There is no silent fallback. A seat whose plane is unreachable, or whose credential proves another identity, refuses to start and names why.
4. Repo-less seats and Antigravity each get an explicit disposition, with the reason in the PR body.
5. #692's MC/KB-specific resident envelope retires with this ticket rather than becoming a durable local target. Its NL/GW placement stays.
6. Institution's "local" choice leaves the agent config, in a neo-agent-institution follow-up once this contract lands.

### Mechanism: claim-time survey (Vega, 2026-10-01, Brain dev `5041af0`)

- **The plane.** In `plane-attach` and `attach`, it is `fleet.planeBase`. Its MC and KB URLs derive the way `FleetTenantService.resourcesFor` derives them (`/mc/mcp`, `/kb/mcp`). Today a resident row (`mcpTarget: null`) becomes `target: 'resident'` and `transport: 'stdio'` (`managedAgentWorkspacePlan.mjs:176-177`). After this ticket it resolves to that plane at start.
- **The credential class follows the plane's `auth.mode`** (`learn/agentos/tooling/MemoryCoreMcpAuth.md`):
  - `github-pat` / `gitlab-pat` (the local dockerized plane) accept a provider PAT. **The Fleet never presents a seat's checkout PAT as plane authority**: `FleetTenantService.probeSeatCredential` says so explicitly, because a checkout PAT carries repo-write scope that a plane must not hold. A seat therefore needs a plane credential of its own; on a provider-PAT plane, that is a second, identity-only PAT. **Today at most one seat can use a given plane.** The tenant store keeps one credential per endpoint (`tenantIdFor` derives the id from host and endpoint digest), and the probe requires that credential to resolve to the seat. The plane credential must become per seat before every seat can join.
  - `seat-token`: verified by `AuthService` against a registry (`ai/mcp/server/shared/helpers/seatToken.mjs`). **No production minter exists:** `mintSeatToken` has no caller outside specs, although the helper's doc names "the seat-config generator" as the minter. Seats on a seat-token plane therefore need the Fleet to become that minter, which it can only do where it owns the registry file. The packaged FM places `NEO_AUTH_SEAT_TOKEN_REGISTRY_PATH` under its own data root, so that is own mode.
  - A connected tenant keeps its stored plane bearer (`FleetTenantService`).
  - `local-bearer` carries no user identity, so no seat can be proven on it, and the start refuses.
  - Whatever the class, `probeSeatCredential({expectedIdentity})` proves it before spawn. The class is never inferred from the plane's name.
- **Own mode serves no MC or KB endpoint today.** The packaged FM starts only two Brain children, the orchestrator (`ai/daemons/orchestrator/daemon.mjs`) and the Fleet (`devFleetServer.mjs`); see Institution `harness/main.mjs` `bootProductBrain`. The orchestrator names `mc-server` only as a Docker service it diagnoses. Recommendation: `own` stays the named exception here (Fix 1). Serving MC and KB over HTTP from the packaged FM would add two server children plus a seat-token minter, and that is the own plane's lane, not this one.

### Decision and build state (Vega, 2026-10-01)

**Decided (Tier 2): (a), a per-seat plane credential.** It lives in `FleetTenantService`, keyed by (plane endpoint, seat), and is stored only after a probe proves it is the seat's own. On a provider-PAT plane this is an identity-only PAT that the operator creates per seat. (b), plane-minted seat tokens, stays with the own-plane lane.

**Built in PR #728:**
- Placement at start (`resolveSeatPlaneTarget`).
- The stored credential, bound to the served `plane.id` and `plane.dataRoot` and proven again at every start (Euclid's read, ADR 0041 §2.4).
- Refusals that name their reason.
- Wake arming for plane seats.
- The credential verb `setPlaneCredential`.

Dispositions on the PR: own mode stays the named exception, and repo-less seats and Antigravity refuse on a Fleet that serves a plane. The Institution prompt for `setPlaneCredential` must ride the same Brain pin.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Wire verb `setPlaneCredential({id, credential})` | `src/fleet/contract/wire.mjs` (`FLEET_WIRE_METHODS`); `fleetLaunchContract.mjs` (`FLEET_CREDENTIAL_METHODS`); `fleetServerPolicy.mjs` (`awaiting-c1`, `lifecycle-write`); `FleetManager.setPlaneCredential` | The credential arrives once and is never returned. The caller names only the seat. The plane is `FleetManager.planeBase`, and the identity to prove is the row's `githubUsername`. Returns `{status: 'stored', endpoint, agentId}` or `{status: 'rejected', reason}`, from a closed vocabulary. | No plane: "this Fleet serves no plane". Unknown seat: "unknown agent". A shell without the Institution projection fails closed with "malformed public intent". | `OwnAgentTeam.md` | `FleetManager.spec`; the wire arm in `FleetTenantService.spec` |
| Stored seat plane credential | `FleetTenantService.storeSeatPlaneCredential`, the sole writer; `seat-plane-credentials.enc` holds `{endpoint: {agentId: {credential, plane: {id, dataRoot}}}}` (AES-256-GCM, `0600`) | Written only after the probe proves the seat's identity and MC and KB both name one served plane. Read only by `resolveSeatPlaneCredential`, which is Brain-internal and never on the wire. | A corrupt store refuses the write and stays byte-identical; a read returns `null`. | — | The seat plane credential arms in `FleetTenantService.spec` |
| Start guard | `startAgentProvisioned`, `resolveSeatPlaneTarget`, `FleetTenantService.probeSeatPlaneCredential` | Placement is resolved at every start, and structural refusals come before any secret is read. A plane seat presents its stored credential, re-proven before checkout: identity, served `plane.id` and `plane.dataRoot`. Its MC and KB get no resident env. | Refuses with a named reason. There is no local fallback. | `OwnAgentTeam.md` | The plane arms in `startAgentProvisioned.spec`, including AC-1 over the real planner |
| Wake arming of a plane seat | `armFleetSeatWake` | Subscribes with the seat's stored plane credential. | `unarmed`, with "the seat holds no plane credential to subscribe with" | — | `armFleetSeatWake.spec` |
| Prompt and public intent (Institution) | neomjs/neo-agent-institution#411 (PR neomjs/neo-agent-institution#413) | The shell projects the verb to `{id}` and prompts in main; the card offers "Plane credential · Set". | On a pin without the verb, the card says the Fleet does not take plane credentials yet. | — | That PR's specs |

## Acceptance Criteria

- [ ] In `plane-attach` and `attach`, no Fleet-prepared seat config, for any launchable harness type, carries a local stdio `memory-core` or `knowledge-base` entry. Red-first spec over the plan and the preparer.
- [ ] In `own`, seats use the own plane's served endpoints, or `own` is named as the exception with its reason on the PR.
- [ ] A seat's plane credential is the class its plane accepts, and it is proven to resolve to the seat's identity before spawn.
- [ ] A seat whose plane is unreachable, or whose credential proves another identity, refuses to start with that reason. There is no local fallback.
- [ ] Repo-less seats and Antigravity each have a stated disposition.
- [ ] `learn/agentos/OwnAgentTeam.md` and the Fleet docs no longer describe a local MC/KB option.
- [ ] *(Post-merge, L4, installed FM; deferred to #571, it needs a pin carrying this and Institution #411)* A newly added seat's first memory is readable from a peer seat on the same plane.

## Out of Scope

- The Memory Core server's own stdio transport (the `Server.mjs` stdio branch), which is still used outside the Fleet.
- Local stdio servers other than MC and KB, and their placement (#692).
- Recovering memories written by resident seats. That is #669's AC-6 receipt.
- Institution's selector (the follow-up above).

## Avoided Traps

- **Keeping "local" as a hidden fallback when the plane is down.** That is the silent fork this removes.
- **Treating `own` mode as a private fork.** It is a supported plane; the defect is reaching it through per-seat processes.
- **Assuming every plane accepts a seat's GitHub PAT.** Tenants carry their own plane bearer.

## Decision Record impact

Aligned with ADR 0019 (§10.7 per-profile placement; `fleet.planeBase` read at the use site). Adjacent to neo `#15805`, which excluded FM-managed generation. No ADR is amended.

## Related

#571 (parent; owns the installed AC-7) · neo-agent-institution #411 / PR #413 (the prompt and public-intent transition; merges before any pin carrying this) · #669 / PR #692 (placement; the MC/KB resident envelope retires here) · PR #698 (stacked on #692) · #79 (Fleet-side wake arming) · Institution `#347` (own mode) · neo `#15805` (adjacent precedent) · neo-agent-institution `#245` (add-agent form)

Live latest-open sweep at 2026-10-01T15:21Z: latest 20 open Brain issues and latest 10 open Institution issues. Keyword searches ("resident mcpTarget", "stdio memory-core resident", "resident target tenant default plane") and a KB ticket query found no equivalent; the nearest is #669's resident half. A2A in-flight sweep (last 30 messages, ~15:22Z): no claim on this scope. Memory Core rationale sweep: no prior decision beyond `#15805`. Body revised at 15:46Z with Euclid's peer read (MESSAGE:8213cc38): own mode, credential classes, and the #15805 boundary.

unowned-rationale: Euclid will not open an overlapping branch while #692's placement repair is active (15:41Z). Vega takes this after #79 if it is still unclaimed.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207
Retrieval Hint: "Fleet resident stdio Memory Core KB target, own mode plane, plane credential class" · `managedAgentWorkspacePlan transport resource stdio` · `resolveProductBrainPlan`







## Timeline

- 2026-10-01T15:23:14Z @neo-opus-vega added the `enhancement` label
- 2026-10-01T15:23:14Z @neo-opus-vega added the `ai` label
- 2026-10-01T15:23:15Z @neo-opus-vega added the `refactoring` label
- 2026-10-01T15:23:15Z @neo-opus-vega added the `architecture` label
- 2026-10-01T15:23:18Z @neo-opus-vega added parent issue #571
- 2026-10-01T15:24:53Z @neo-opus-ada cross-referenced by PR #692
- 2026-10-01T15:32:51Z @neo-opus-grace cross-referenced by #700
- 2026-10-01T15:47:01Z @neo-gpt cross-referenced by #669
- 2026-10-01T16:22:57Z @neo-opus-vega cross-referenced by #79
- 2026-10-01T16:37:27Z @neo-gpt cross-referenced by #571
- 2026-10-01T17:37:33Z @neo-gpt-emmy cross-referenced by PR #705
- 2026-10-01T18:24:22Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T19:14:47Z @neo-opus-vega referenced in commit `57cfd3f` - "feat(fleet): each seat keeps its own plane credential, stored only once the plane proves it is the seat (#699)

The tenant store holds one bearer per plane endpoint, and the seat probe
admits that bearer for one identity only, so at most one seat could use a
plane. FleetTenantService now keeps a second encrypted store,
seat-plane-credentials.enc, holding {endpoint: {agentId: credential}}.
storeSeatPlaneCredential probes the credential against the plane before
anything persists, and refuses one that resolves to another identity, a
rejected bearer or an unreachable plane. resolveSeatPlaneCredential is
Brain-internal and never on the wire. The strict encrypted read is shared by
both stores, and the seat id reuses the one seat-segment rule."
- 2026-10-01T19:20:02Z @neo-opus-vega referenced in commit `1562f88` - "feat(fleet): a seat's Memory Core and Knowledge Base resolve to the plane the Fleet serves (#699)

First slice of #699, with no consumer yet. resolveSeatPlaneTarget decides
where a seat's MC/KB live at each start:
- a connected tenant is kept;
- any other seat on a Fleet with fleet.planeBase reaches that plane;
- own mode (no plane) and harnesses that cannot reach a remote Memory Core
  keep the per-seat store as named exceptions;
- a declared plane that is not a secure MCP endpoint refuses rather than
  fork memory into a private store.
The two resource URLs beneath an endpoint move into mcpWireParsing as
planeMcpResources, replacing FleetTenantService's private copy."
- 2026-10-01T19:20:02Z @neo-opus-vega referenced in commit `b274eeb` - "feat(fleet): each seat keeps its own plane credential, stored only once the plane proves it is the seat (#699)

The tenant store holds one bearer per plane endpoint, and the seat probe
admits that bearer for one identity only, so at most one seat could use a
plane. FleetTenantService now keeps a second encrypted store,
seat-plane-credentials.enc, holding {endpoint: {agentId: credential}}.
storeSeatPlaneCredential probes the credential against the plane before
anything persists, and refuses one that resolves to another identity, a
rejected bearer or an unreachable plane. resolveSeatPlaneCredential is
Brain-internal and never on the wire. The strict encrypted read is shared by
both stores, and the seat id reuses the one seat-segment rule."
- 2026-10-01T19:21:07Z @neo-opus-vega referenced in commit `6ecfaca` - "feat(fleet): one injected FleetManager.planeBase serves both wake arming and a seat's start (#699)

#705 injected the attached plane through wakeStateOptions for wake arming,
and the seat's start now needs the same leaf. FleetManager.planeBase is
injected once by devFleetServer in plane mode (null in host mode), like
managedRoot. armSeatWake reads it, and startAgent forwards it to the start
composer."
- 2026-10-01T20:11:35Z @neo-opus-vega referenced in commit `9a4a4bc` - "feat(fleet): a seat's Memory Core and Knowledge Base resolve to the plane the Fleet serves (#699)

First slice of #699, with no consumer yet. resolveSeatPlaneTarget decides
where a seat's MC/KB live at each start:
- a connected tenant is kept;
- any other seat on a Fleet with fleet.planeBase reaches that plane;
- own mode (no plane) and harnesses that cannot reach a remote Memory Core
  keep the per-seat store as named exceptions;
- a declared plane that is not a secure MCP endpoint refuses rather than
  fork memory into a private store.
The two resource URLs beneath an endpoint move into mcpWireParsing as
planeMcpResources, replacing FleetTenantService's private copy."
- 2026-10-01T20:11:35Z @neo-opus-vega referenced in commit `8868e5c` - "feat(fleet): each seat keeps its own plane credential, stored only once the plane proves it is the seat (#699)

The tenant store holds one bearer per plane endpoint, and the seat probe
admits that bearer for one identity only, so at most one seat could use a
plane. FleetTenantService now keeps a second encrypted store,
seat-plane-credentials.enc, holding {endpoint: {agentId: credential}}.
storeSeatPlaneCredential probes the credential against the plane before
anything persists, and refuses one that resolves to another identity, a
rejected bearer or an unreachable plane. resolveSeatPlaneCredential is
Brain-internal and never on the wire. The strict encrypted read is shared by
both stores, and the seat id reuses the one seat-segment rule."
- 2026-10-01T20:11:35Z @neo-opus-vega referenced in commit `fa6a7c7` - "feat(fleet): one injected FleetManager.planeBase serves both wake arming and a seat's start (#699)

#705 injected the attached plane through wakeStateOptions for wake arming,
and the seat's start now needs the same leaf. FleetManager.planeBase is
injected once by devFleetServer in plane mode (null in host mode), like
managedRoot. armSeatWake reads it, and startAgent forwards it to the start
composer."
- 2026-10-01T20:11:36Z @neo-opus-vega referenced in commit `161b983` - "feat(fleet): a seat starts against the plane the Fleet serves, with its own plane credential bound to the plane that proved it (#699)

On a Fleet that serves a plane, a seat's Memory Core and Knowledge Base are
that plane's: startAgentProvisioned resolves the placement, presents the
seat's own stored plane credential, never its checkout PAT, and proves it again
before any checkout, on the plane it was stored against (served plane.id and
plane.dataRoot, ADR 0041 §2.4). A seat that cannot get there refuses and says
why: no managed repo, a harness with no remote Memory Core, no stored
credential, or a failed proof. A private per-seat store is no fallback; only a
Fleet that serves no plane keeps the per-seat servers.

Wake arming subscribes plane seats with the same credential, and the wire
gains setPlaneCredential, a credential-bearing verb like connectTenant."
- 2026-10-01T20:11:38Z @neo-opus-vega cross-referenced by PR #728
- 2026-10-01T20:15:08Z @neo-opus-vega cross-referenced by #411
- 2026-10-01T20:42:37Z @neo-opus-vega referenced in commit `f2658ac` - "fix(fleet): wake arming proves the stored plane binding before it subscribes, so a Start on a running seat cannot re-arm on another plane (#699)

FleetManager.startAgent arms after every Start, and a Start on a seat that is
already running returns before the start's own plane proof. Arming then used the
stored credential after an identity check alone, so a plane recreated behind the
same URL could take a subscription and a published route on the old credential.
Arming now calls probeSeatPlaneCredential with the stored plane.id and data root
before it creates the client; a mismatch is unarmed with the reason, and nothing
is subscribed or published. OwnAgentTeam.md says where the proof runs and
qualifies the plane credential as the MC/KB one, beside the checkout PAT."
- 2026-10-01T20:45:44Z @neo-opus-vega cross-referenced by PR #413
- 2026-10-01T21:06:59Z @tobiu referenced in commit `783b35b` - "feat(fleet): a seat starts against the plane the Fleet serves, with its own plane credential bound to the plane that proved it (#699) (#728)

* feat(fleet): a seat's Memory Core and Knowledge Base resolve to the plane the Fleet serves (#699)

First slice of #699, with no consumer yet. resolveSeatPlaneTarget decides
where a seat's MC/KB live at each start:
- a connected tenant is kept;
- any other seat on a Fleet with fleet.planeBase reaches that plane;
- own mode (no plane) and harnesses that cannot reach a remote Memory Core
  keep the per-seat store as named exceptions;
- a declared plane that is not a secure MCP endpoint refuses rather than
  fork memory into a private store.
The two resource URLs beneath an endpoint move into mcpWireParsing as
planeMcpResources, replacing FleetTenantService's private copy.

* feat(fleet): each seat keeps its own plane credential, stored only once the plane proves it is the seat (#699)

The tenant store holds one bearer per plane endpoint, and the seat probe
admits that bearer for one identity only, so at most one seat could use a
plane. FleetTenantService now keeps a second encrypted store,
seat-plane-credentials.enc, holding {endpoint: {agentId: credential}}.
storeSeatPlaneCredential probes the credential against the plane before
anything persists, and refuses one that resolves to another identity, a
rejected bearer or an unreachable plane. resolveSeatPlaneCredential is
Brain-internal and never on the wire. The strict encrypted read is shared by
both stores, and the seat id reuses the one seat-segment rule.

* feat(fleet): one injected FleetManager.planeBase serves both wake arming and a seat's start (#699)

#705 injected the attached plane through wakeStateOptions for wake arming,
and the seat's start now needs the same leaf. FleetManager.planeBase is
injected once by devFleetServer in plane mode (null in host mode), like
managedRoot. armSeatWake reads it, and startAgent forwards it to the start
composer.

* feat(fleet): a seat starts against the plane the Fleet serves, with its own plane credential bound to the plane that proved it (#699)

On a Fleet that serves a plane, a seat's Memory Core and Knowledge Base are
that plane's: startAgentProvisioned resolves the placement, presents the
seat's own stored plane credential, never its checkout PAT, and proves it again
before any checkout, on the plane it was stored against (served plane.id and
plane.dataRoot, ADR 0041 §2.4). A seat that cannot get there refuses and says
why: no managed repo, a harness with no remote Memory Core, no stored
credential, or a failed proof. A private per-seat store is no fallback; only a
Fleet that serves no plane keeps the per-seat servers.

Wake arming subscribes plane seats with the same credential, and the wire
gains setPlaneCredential, a credential-bearing verb like connectTenant.

* fix(fleet): wake arming proves the stored plane binding before it subscribes, so a Start on a running seat cannot re-arm on another plane (#699)

FleetManager.startAgent arms after every Start, and a Start on a seat that is
already running returns before the start's own plane proof. Arming then used the
stored credential after an identity check alone, so a plane recreated behind the
same URL could take a subscription and a published route on the old credential.
Arming now calls probeSeatPlaneCredential with the stored plane.id and data root
before it creates the client; a mismatch is unarmed with the reason, and nothing
is subscribed or published. OwnAgentTeam.md says where the proof runs and
qualifies the plane credential as the MC/KB one, beside the checkout PAT."
- 2026-10-01T21:07:00Z @tobiu closed this issue
### @neo-gpt-emmy - 2026-10-03T11:23:55Z

## Operator correction: the added PAT burden was not authorized

The installed Mnemosyne onboarding attempt exposed a broken journey: Add Agent accepted the seat's PAT and published its card, but Start refused because a separate plane-credential slot was empty. The only visible recovery was a buried prompt instructing the operator to create another identity-only PAT. The operator explicitly rejected that requirement on 2026-10-03.

The operator input recorded at the top of this issue requested shared-plane MC/KB. The later second-PAT requirement is recorded here as an agent-selected Tier-2 decision; it is not the operator's instruction. I approved the Brain implementation in #728 and described the separate credential as sound. That review failed to challenge the operator-facing journey.

The required product outcome is one credential entry for an ordinary seat on the connected Agent OS, with identity and served-plane verification preserved, and no hidden setup left after Add Agent. Distinct authentication requirements for an explicitly different service must be surfaced within that journey when they actually apply, rather than imposed on every local-plane seat. A second human-created PAT must not be a prerequisite for this pilot.

The adjacent URL-choice failure is verified too: the UI offers the saved connection, while the registry rejects it as already assigned to Sophie. The prior target consequently remains selected. That is an offered-but-unusable choice, not evidence that the operator failed to configure the seat.

I am carrying the correction with the UI and onboarding owners; the current copied Mnemosyne profile and its memory remain preserved. No identity/proof guard is being removed.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16

- 2026-10-03T11:41:50Z @neo-gpt-emmy cross-referenced by #809
- 2026-10-03T11:48:35Z @neo-gpt-emmy cross-referenced by #503

