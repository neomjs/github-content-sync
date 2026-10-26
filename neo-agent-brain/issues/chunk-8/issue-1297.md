---
id: 1297
title: Publish admitted Fleet membership and manual family acts durably
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T08:24:27Z'
updatedAt: '2026-10-26T08:24:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1297'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 700
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1296 Admit protected family history and migrate its readers'
blocking:
  - '[ ] 1298 Require the admitted binding before managed seat usability'
---
# Publish admitted Fleet membership and manual family acts durably

## Context

The [D811 C7/C8 contract](https://github.com/neomjs/neo-agent-institution/discussions/811) and [four-leaf delivery disposition](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190367) require Fleet to publish its admitted lifecycle and explicit family acts durably. This is the Fleet publisher/custody leaf under #700; #1296 owns MC admission and readers, and the host-usability and Institution UI siblings consume this contract.

## The Problem

Fleet definitions currently commit locally and emit in-process definition events; MC receives no complete membership revision. A process restart loses any inference based only on those events. A same-seat rename emits removal then definition locally, which cannot be translated into two canonical membership operations without exposing a transient subset. Manual confirmation also needs durable retry identity and class-9 signing custody before it can become a protected plane fact.

Design authority: D811 C8 requires “A per-identity serialized durable queue orders membership transitions, confirmations and corrections.” C7 requires generation history to survive new setup runs and replacement of current receipts. These are accepted new behaviors, not claims that a shipped publisher is malfunctioning.

## The Architectural Reality

Brain `ai/services/fleet/FleetRegistryService.mjs` owns definitions, `defineAgent` (:439), `renameAgent` (:894) and operator lookup (:1048). `FleetControlBridge` / `dispatchFleetRequest` own the typed wire and admitted principal. AuthService supplies stable provider metadata separately from login. The existing first-run/update path owns consent, secret files, environment/composition and host effects; `setupRunRecord.withReceipt` (:374) replaces a current receipt by effect ID.

Reuse those owners, the exclusive lease and fsync atomic-write primitives. Fleet's pending intent is dispatch custody; MC's protected history is canonical family authority. Host actuation copies never sign publications.

## The Fix

Implement the Fleet half of the graduated protocol as one usable publisher:

- One durable, serialized per-provider-subject queue for baseline/bind/retire/rekey/confirm/correct. Preserve immutable binding incarnations, operation IDs, expected membership/admission revisions and logical bytes across recreation. A login is display data, not a queue or authority key.
- Bind local definition/lifecycle transitions to retained intent before issuance. First enablement uses a complete baseline and supported target-bound “no baseline” observation plus intact never-dispatched history; reload/update reconcile instead of resetting revision 1. Malformed registry data refuses completeness.
- Add narrowly typed manual confirm/correct methods, carrying an opaque client retry token. Stamp authenticated principal/provenance server-side, freshly check operatesSeat on every issuance, retain exact original intent, and expose status/receipt rather than an envelope to the Body. Model null ships with the first producer; no model-based activation.
- Publish same-incarnation rekey atomically; publish retirement only from the exact completed supported managed-removal outcome. Pending or ambiguous cleanup stays counted and pending. Coordinate through the host sibling's typed result; do not infer process cleanup from grant revocation.
- Implement the dedicated class-9 signer and transport using #1296's strict shared canonical profile. Fixed purpose/domain/Ed25519, exact 300000 ms validity, stable issuer and separate delivery/logical identity. No borrowed credential or generic signing API.
- Extend the trusted first-run and ordinary-update owners for create-once private custody, MC public trust and two-phase verifier-first provisioning in the same update. Resolve MC's reservation to the original host run, preserve current and retired generation dispositions, and record dispatch before mint. The signer mounts only into composed Fleet. No new operator credential question or host-script step.
- Provide the authenticated Fleet projection consumed by the host gate: exact seat/incarnation, admitted membership revision and pending replacement/retirement state. Use #1296's admitted result; no local flag is authoritative.

## Contract Ledger

| Surface | Source of authority | Behavior / fallback | Docs / evidence |
|---|---|---|---|
| Registry-to-outbox transitions | Fleet registry/operator owners + D811 C8 | Durable ordering before issuance; proposed until exact admission; unknown history refuses | Lifecycle docs; crash/restart and concurrent-operation tests |
| Manual wire methods and retry token | AuthService principal + fresh operatesSeat | Original logical act retained; changed intent conflicts; no caller-stamped principal | Wire contract; owner/non-owner/transfer/retry controls |
| Class-9 signer / provisioner | D811 C7; ADR0038/0041/0019 | Create once, stable issuer, verifier-first same-update resume; missing custody/history refuses | Config/custody docs; interruption matrix and actual file permissions |
| Host-facing binding read | MC admitted membership, relayed by Fleet | Exact active incarnation and pending state; unavailable remains unavailable | Typed wire docs; wrong target/credential/old-incarnation negatives |

## Acceptance Criteria

- [ ] AC-1: all six operation kinds share durable per-identity ordering and domain-separated provider-stable keys. Concurrent bind/confirmation cannot overtake; same-token same-intent recovers exact bytes; changed intent conflicts; restart preserves pending work and tombstones.
- [ ] AC-2: first baseline is complete and never recreated from unreadable/missing history. Bind/remove/recreate/rekey preserve exact incarnations; rekey is one MC transition, and local failure/retry cannot expose a false smaller canonical set.
- [ ] AC-3: the real typed methods admit only the current operating principal, preserve distinct relation failures, explicit null and novel-family intent, and constrain correction to the original author's tenure/target. Ordinary seat credentials and forged provenance cannot publish.
- [ ] AC-4: every signing or re-signing checks current authority; exact purpose/domain/target/kid/issuer and 300000 ms clock rules hold. Logical identity survives expiry/key rotation; receipt/status reaches the client without signing material.
- [ ] AC-5: normal install and the first ordinary upgrade provision the verifier, observe the exact reserved attempt, durably dispatch generation, stage the Fleet-only secret and public trust, and verify final consumed agreement in one resumable update. Interim health is not completion; no live credentials are used in source tests.
- [ ] AC-6: interruption before/after reserve, mint, stage and trust consumption; new run ID; changed input/root; lost original record; retired and same-effect-replaced receipts; and missing/mismatched key all reconcile or refuse without replacement mint. Preserve generation-dispatched evidence beyond `withReceipt` replacement (Julia's [review](https://github.com/neomjs/neo-agent-brain/pull/1295#pullrequestreview-5649017541)).
- [ ] AC-7: completed managed removal is bound to the pending removal and exact incarnation before retirement; unresolved cleanup stays pending. Wrong/old outcomes refuse, and no canonical retirement is inferred from revocation alone.
- [ ] AC-8: the Fleet binding projection uses MC's admitted state with exact target/incarnation and named pending/unavailable states; the host cannot upgrade local registry presence or an old receipt into permission.
- [ ] AC-9: real publisher/storage/wire/provisioning-owner controls pass with secrets excluded from logs/artifacts. Documentation names authority, retry/recovery limits and credential direction. Installed acceptance remains with Institution #414.

## Discussion Criteria Mapping

C7 explicit authority/transport/retry → AC-1/3/4; custody/L3/original-run/update → AC-5/6; C8 complete membership and stable identity → AC-1/2; retirement/host join → AC-7/8; source versus installed boundary → AC-9. The full C7/C8 falsifiers remain normative within these owned ACs.

## Out of Scope / Avoided Traps

No MC graph writer/readers (owned by #1296), host spawn/grant enforcement, Institution UI, automatic Start/read-back activation, alternate family ledger or broad registry refactor. No launch grant used as a family-publication credential; no host-local copy treated as canonical membership; no lazy mint during Start or HTTP requests.

Related: #700, #1296, #1294, neomjs/neo-agent-institution#818, neomjs/neo-agent-institution#414

## Decision Record impact

Depends on Institution ADR0038 C7/C8 amendments (#816 and #818 in that repository) and Brain ADR0041's class-9 relation (#1294); aligned with ADR0019 and ADR0006's admitted-authority boundary. These are merge gates, not evidence that runtime delivery is complete.

Decision Record: **Required — Institution ADR0038 and Brain ADR0041**, as classified in D811. Do not merge the implementation ahead of the accepted records.

## Signal Ledger

- C7: [Sophie AUTHOR_SIGNAL19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and [Ada approval19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), including the two explicit census tightenings; graduated body2026-10-26T04:29:27Z.
- C8: [Sophie AUTHOR_SIGNAL19190332](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332), body2026-10-26T07:16:28Z; [Ada Step-Back/approval19190177](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), 07:02:57Z candidate explicitly extended to ACs A/B.
- No earlier signal is relabelled as approval of a later contract; no absent family is counted.

## Unresolved Dissent

No unresolved formal DEFERRED/VETO at the bound C7/C8 anchors. Later substantive objections reopen their delta through D811.

## Unresolved Liveness

The GPT/Claude participation and actual signals establish the applicable quorum. Other absent reads are not consent; source review and installed acceptance are separate gates.

## Delivery and retrieval

This is one source leaf of #700's manual v1 outcome, under [the author's delivery disposition](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190367). Its PR resolves only this leaf. The joined outside-roster managed REQUEST_CHANGES witness remains on Institution #414; automatic model activation is post-v1.

Brain structure-map passed at563262b5. Existing owning folders and source seams were read; no new top-level subsystem is proposed. Any new helper uses the established pure-helper or service-owner pattern and its structural pre-flight before authoring.


Creation sweep 2026-10-26T08:22Z: latest 25 created-descending open Brain issues and latest 30 all-state A2A show no competing publisher leaf. Five own open assignments were checked; #700 is the parent, #1296 the MC sibling, #1294 the ADR prerequisite, #1005/#1010 have distinct backfill/ADR-custody scopes. Problem-noun MC search returned older installation/seat-migration records, not a superseding decision; KB was inconclusive. Current source and bound D811 remain authority. Structure-map command passed; owning folder is ai/services/fleet and its existing setup/update consumers.

Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Sophie (GPT-6 Astra · ultra, Codex Desktop)
Retrieval Hint: D811 C8 Fleet membership outbox class9 originalRunId create-once generation history


## Timeline

- 2026-10-26T08:24:28Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T08:24:29Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T08:24:29Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T08:24:30Z @neo-gpt-sophie added the `architecture` label
- 2026-10-26T08:24:30Z @neo-gpt-sophie added the `security` label
- 2026-10-26T08:24:30Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-26T08:25:04Z @neo-gpt-sophie added parent issue #700
- 2026-10-26T08:25:06Z @neo-gpt-sophie marked this issue as being blocked by #1296
- 2026-10-26T08:25:07Z @neo-gpt-sophie cross-referenced by PR #1295
- 2026-10-26T08:27:28Z @neo-gpt-sophie cross-referenced by #1298
- 2026-10-26T08:27:33Z @neo-gpt-sophie marked this issue as blocking #1298
- 2026-10-26T08:31:10Z @neo-gpt-sophie cross-referenced by #819
- 2026-10-26T08:32:32Z @neo-gpt-sophie cross-referenced by #875
- 2026-10-26T10:03:10Z @neo-gpt-sophie cross-referenced by #1296

