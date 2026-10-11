---
id: 989
title: Record local hook projection placement across Docker and host
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt
createdAt: '2026-10-11T02:23:08Z'
updatedAt: '2026-10-11T02:45:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/989'
author: neo-gpt
commentsCount: 0
parentIssue: 107
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-11T02:45:59Z'
---
# Record local hook projection placement across Docker and host

## Context

Brain #107's accepted local awareness composition has delivered primitives but no verified host publication. The [placement audit](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6104013184), [independent challenge](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6104491836) and [amendment draft](https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6104559286) bind this work to current source and the selected local Docker profile.

Design authority: ADR 0035 §6 requires revalidation when “the writer owner, shared runtime root, or SQLite/WAL posture changes.” Its §2.6 requires the actual rename to remain fenced at the MC resource.

## The Problem

The accepted July record predates today's container/host placement. At Brain `5d2e2746`, the MC root is a plane member; the deployed canonical-local service's default publication directory lies in its private named volume. The host hook requires a trusted local path and binding. A valid bearer or an MC-written canary alone does not attest a recipient's harness instance.

The gap must be recorded before production registration adopts an unreviewed placement or mistakes namespace agreement for identity proof.

## The Architectural Reality

- ADR 0035 §2.6 owns the MC writer, same-transaction rename and local pure reader; §2.4 owns categorical binding and validated agent-wide fallback.
- ADR 0019 §10.5/§10.7 owns plane-member coherence and profile election; §10.9 distinguishes host source from container target.
- `ai/mcp/server/memory-core/configBase.mjs:638` declares `hookProjectionRoot` as a plane member.
- `AuthService.createSeatTokenVerifier` authenticates a minted agent subject; `Server.buildRequestContext` binds that actor. Neither supplies the missing projection-instance attester.
- ADR 0039 keeps host entrypoints from importing cloud durable stores.

## The Fix

Amend only the two existing ADRs. Record canonical-local MC-written host publication, packaged own-mode's single host namespace, and no local-hook publication grant for cloud/remote profiles. Preserve resource-side fencing, plane-member classification and §2.4's proof/fallback rules. Name the implementation evidence still required instead of claiming it exists. Fold Grace's R1: a fresh launcher-supplied nonce is written by MC at root scope and compared at the declared host source; stale witnesses and target-directory cleanup cannot satisfy placement. Carry §10.7's four re-runs and the existing §6 revalidation for lease/channel schema changes across writer generations.

| Surface | Authority | Recorded behavior | Failure | Evidence |
| --- | --- | --- | --- | --- |
| Local publication root | ADR 0035 §2.6; ADR 0019 §10.7 | MC owns the RW host bind and performs the fenced rename | Missing/divergent placement withholds enrichment | Existing leaf + live mount census; future container→host witness |
| Host source / container target | ADR 0019 §10.9 | Distinct declared namespace contracts, resolved by their owners | No guessed path or silent alias | Future fresh nonce-bound, root-scoped namespace agreement witness |
| Recipient binding | ADR 0035 §2.4 | A canary/registration/bearer never upgrades to instance proof | Existing route-only / validated agent-wide rules | Source context and absent attester census; future foreign-target test |

## Decision Record impact

Amends ADR 0035 §2.6 and ADR 0019 §10.7; preserves their accepted authority split and ADR 0039. This is the record gate before implementation under #107, not a new publisher or permission contract.

ADR successor-risk: adr-amendment-required — the current Docker split postdates ADR 0035's July acceptance; the linked executed audit and independent challenge require the bounded profile amendment before registration.

Decision Record: Required — the two existing ADR amendments are this ticket's deliverable.

## Acceptance Criteria

- [ ] ADR 0035 records the three profile dispositions, keeps MC's fenced rename and pure local hooks, and separates placement witnesses from recipient attestation.
- [ ] ADR 0019's profile election records the host-readable plane-member publication resource and distinct namespace contracts without reclassifying it as a backup escape hatch or adding a competing resolver.
- [ ] The record requires a fresh launcher-supplied nonce, written by MC at root scope and compared at Start, and names host atomic-read, stale-holder takeover, namespace divergence and foreign-target falsifiers as outstanding implementation gates; no source/CI receipt is promoted to installed acceptance.
- [ ] The PR changes only the two ADR files and keeps #107's production registration, launcher attester and live multi-resident acceptance open.

## Out of Scope

Code, Compose changes, new config leaves/API/permission scopes, deployment, remote transport, generated harness settings, online ranking and offline evidence capture.

## Avoided Traps

A host writer under a remote advisory lease does not resource-fence the rename. Copying a container file or fetching from a Stop hook changes the accepted contract. Treating a canary as instance proof hides the still-missing attester. A documentation merge alone cannot close #107.

## Related

Parent: #107
Source: [D#15090](https://github.com/neomjs/neo/discussions/15090), accepted composition; its old signals do not certify a live installation.
Related: ADR 0019 · ADR 0035 · ADR 0039

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b

Retrieval Hint: `hookProjectionRoot container private publication root host hook attestation ADR0035`

Creation sweep at 2026-10-11T02:22Z: latest 20 open Brain issues (number/title/author/labels/URL) and the recent all-state A2A slice found no equivalent amendment or competing claim. Own assignment: six open outcomes; #107 is the matching parent, whose body and current audit were read. Three prior-art queries plus a final problem-noun query returned off-topic rows; they establish no absence claim. Live exact searches found #107 and the distinct runtime-generation #980. The executed structure map covered 93 ai folders; placement belongs in the two existing decision records. KB returned no settled placement answer. No new .mjs file or directory is proposed.

## Timeline

- 2026-10-11T02:23:08Z @neo-gpt assigned to @neo-gpt
- 2026-10-11T02:23:10Z @neo-gpt added the `documentation` label
- 2026-10-11T02:23:10Z @neo-gpt added the `enhancement` label
- 2026-10-11T02:23:10Z @neo-gpt added the `ai` label
- 2026-10-11T02:23:11Z @neo-gpt added the `architecture` label
- 2026-10-11T02:23:39Z @neo-gpt added parent issue #107
- 2026-10-11T02:28:01Z @neo-gpt cross-referenced by #107
- 2026-10-11T02:28:32Z @neo-gpt cross-referenced by PR #990
- 2026-10-11T02:46:00Z @tobiu referenced in commit `dddc1bc` - "docs(agentos): record local hook projection placement (#989) (#990)"
- 2026-10-11T02:46:00Z @tobiu closed this issue
- 2026-10-11T03:21:45Z @neo-gpt cross-referenced by #992
- 2026-10-11T05:36:59Z @neo-opus-grace cross-referenced by PR #996

