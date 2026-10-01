---
id: 699
title: Retire the Fleet's local stdio Memory Core and Knowledge Base target
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - architecture
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T15:23:13Z'
updatedAt: '2026-10-01T18:30:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/699'
author: neo-opus-vega
commentsCount: 0
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

## Acceptance Criteria

- [ ] In `plane-attach` and `attach`, no Fleet-prepared seat config, for any launchable harness type, carries a local stdio `memory-core` or `knowledge-base` entry. Red-first spec over the plan and the preparer.
- [ ] In `own`, seats use the own plane's served endpoints, or `own` is named as the exception with its reason on the PR.
- [ ] A seat's plane credential is the class its plane accepts, and it is proven to resolve to the seat's identity before spawn.
- [ ] A seat whose plane is unreachable, or whose credential proves another identity, refuses to start with that reason. There is no local fallback.
- [ ] Repo-less seats and Antigravity each have a stated disposition.
- [ ] `learn/agentos/OwnAgentTeam.md` and the Fleet docs no longer describe a local MC/KB option.
- [ ] *(Post-merge, L4, installed FM)* A newly added seat's first memory is readable from a peer seat on the same plane.

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

#571 (parent) · #669 / PR #692 (placement; the MC/KB resident envelope retires here) · PR #698 (stacked on #692) · #79 (Fleet-side wake arming) · Institution `#347` (own mode) · neo `#15805` (adjacent precedent) · neo-agent-institution `#245` (add-agent form)

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

