---
id: 1818
title: Read qualified family bootstrap and operation receipts
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-16T18:25:43Z'
updatedAt: '2026-11-16T18:26:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1818'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 1296
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1511 Wire the protected family service through Memory Core'
blocking:
  - '[ ] 1297 Publish admitted Fleet membership and manual family acts durably'
---
# Read qualified family bootstrap and operation receipts

## Context and problem

The Fleet publisher in #1297 needs two observations before it can complete safely: a proved absence of the first baseline, and the exact retained receipt of an operation whose acknowledgement was lost. This is a read-side continuation of #1296, following the merged #1510/#1511 source.

At `d58d950e`, `identityPublicationAdmission.load()` already validates the complete protected population in one physical transaction. Its no-plane-state branch permits only the valid lineage and founding records. The registered snapshot turns that result into `membership-unknown`, without a target/generation-bound absence observation.

The available snapshot also omits operation receipts and their logical digests. Disposable in-memory SQLite probes using the actual admission/snapshot/lineage helpers found:
- Two baselines differing only in operation ID produce identical family snapshots and lineage witnesses.
- Confirmations with the same admission ID but a different valid `unattestedReason` produce identical snapshots and different retained logical digests.
- The exact original carrier refuses at expiry as `outside-validity`; admission checks current trust and validity before receipt lookup.

These are source counterexamples, not an observed live loss. Remaining pending after an authority transfer is safe; reconstructing an exact receipt from a family projection is unsupported.

## Architectural reality

Design authority: #1297 AC-2 requires a supported target-bound no-baseline observation plus intact never-dispatched Fleet history. Institution `adr:fm-client-topology` §2.5.3 requires the publisher to retain the actual MC receipt before marking an intent accepted, with fresh authority on every issuance/re-signing. No read grants issuance authority.

The existing owners are:
- `ai/services/memory-core/helpers/identityPublicationAdmission.mjs`: physical transaction, complete census and retained receipt validation.
- `GraphService.readIdentityPublicationSnapshot(options)` and `IdentityPublicationService.getIdentityPublicationSnapshot()`: protected owner and boot-generation composition.
- `get_identity_publication_snapshot` in MC `toolService.mjs` / `openapi.yaml`: authenticated registered read.
- `ai/services/shared/identityPublicationClient.mjs`: independent credential/identity/plane proof and strict detached result validation.

Prescription checked: expose the already-validated physical observation through these owners. A Fleet-local mirror, raw graph lookup or second uncoordinated lineage read cannot prove the same fact.

## Intended change and contract ledger

Extend the existing registered read with an optional bounded `operationId` selector, using the canonical publication identifier grammar. The server still selects issuer, trust, plane/root and clock. No new MCP tool, signing operation or config leaf.

Add a closed version-1 `publicationObservation` beside the existing result:
- `target`, current publication `revision` (zero only for proved baseline absence), and `baseline: absent | present`;
- consumed `lineage: {issuer, reservation, head, fingerprint}` and `founding: {revision, digest}`, from that same physical transaction;
- `receipt`: `{state: not-requested}`, `{state: not-found, operationId}`, or `{state: found, operationId, digest, result}`, where `result` is the exact existing stored receipt.

Emit it only after complete authority/fence/founding/history validation. Before a baseline, ordinary family state remains `unavailable` / `membership-unknown`; after a baseline, the existing `snapshot` shape is unchanged.

| Surface | Authority | Behavior / refusal | Evidence and docs |
| --- | --- | --- | --- |
| Physical snapshot owner | Current consumed lineage, clear fence and complete protected census | Select absence/receipt in the same read transaction; corrupt/orphan rows never become not-found | Actual SQLite controls; IdentitySchema |
| Registered read input/output | Existing MC authentication and boot owner | Optional bounded operation selector only; no caller target/issuer/trust/time; legacy empty-input reads remain compatible | Real ToolService/OpenAPI path and negative input controls |
| Publisher client `readPublicationObservation({operationId?, expectedLineage})` | Independently held expected target and consumed-lineage expectation | Require existing identity/target proof, exact lineage match and strict closed observation validation; source failure stays unavailable | Registered-client composition, reconnect/wrong-target/generation/malformed controls |
| Existing `readSnapshot()` | Existing family-reader contract | No positive family result before baseline; no interpretation of a missing observation as empty history | Existing client/consumer regression suites |

The publisher compares a found digest with its retained canonical logical intent. Neither not-found nor baseline absence replaces its intact original journal, current authority or complete capture.

## Acceptance criteria

- [ ] AC-1: A bound, unfenced, complete lineage+founding-only store yields a target/generation-bound baseline-absent observation at revision zero. Missing founding, orphan protected nodes/edges, wrong target, active transaction, unsupported trust or fence never yield absence.
- [ ] AC-2: For each of the six operation kinds, a selected accepted operation returns its exact original digest and receipt. A genuinely missing operation returns not-found only after the complete same-transaction census. Malformed or missing retained rows are unavailable, never partial success.
- [ ] AC-3: Exercise lost acknowledgement followed by carrier expiry and operator transfer: read the original receipt without issuing or re-signing an operation. Changed retained logical bytes do not match. The read writes nothing and grants no new issuance or correction authority.
- [ ] AC-4: The real registered read and client enforce the optional selector and closed observation shape, retain identity/plane/root checks on reconnect, and refuse a mismatched expected lineage. No generic graph read or second independently sampled lineage read supplies authority.
- [ ] AC-5: Existing no-argument family readers retain their exact available snapshot and unavailable-before-baseline behavior. Missing observations, unsupported versions and malformed fields cannot become successful emptiness.
- [ ] AC-6: Physical-owner, registered-service/client and affected source-consumer controls, OpenAPI parity and Brain preflight/CI pass. Record the counterexamples and explicit remedies; no live provisioning or installed acceptance is claimed.

## Scope and authority

One source PR under #1296; a concrete prerequisite for #1297. The publisher journal/signer/provisioner, host enforcement #1298, D825 relation/cutover and trust rotation mutations remain outside this read change. Rotation remains an unresolved delivery obligation of the accepted C7/#1297 contract; the current `lineage-rotation-required` refusal is not rotation completion.

Decision Record: Required — the accepted Institution ADR0038 C7/C8 and Brain ADR0041 records remain the authority. Decision Record impact: aligned with neomjs/neo-agent-institution#adr:fm-client-topology and neomjs/neo-agent-brain#adr:bootstrap-record-verified-plane-handoff; no amendment.

Discussion criteria mapping: this delivers the source observation needed by C7 receipt-backed retry and C8 first-baseline reconciliation. It changes no six-operation grammar, signing purpose, temporal policy or cutover authority. Quorum/version anchors remain in parent #1296 and publisher #1297.

Avoided traps: treating generic unavailability as absence; deriving exact acceptance from projected family state; bypassing expiry/authority to obtain an idempotent receipt; introducing a second ledger; combining separate reads into an alleged coherent observation.

Owner: Sophie, continuing the already-owned protected-source stream. No newcomer labels: this is an active security-sensitive source dependency.

Structure map passed at `d58d950e`; existing owner/helper/client folders above, no new subsystem. Live latest-open/closed and all-state A2A sweeps on 2026-11-16: latest 20 created-descending open issues, 157 open (red band), closed family-publication hits #1509/#1510/#1511 scope and exclusions read; no equivalent continuation. Nine own assignments checked, with #1296/#1297/#1298 bodies read. MC/KB problem-noun searches returned unrelated/older records, not an equivalent decision. This folds into the existing source epic rather than creating a parallel outcome.

Origin Session ID: ab8e3724-e3a3-4901-9bd5-cacaf02e81ac
Authored by Sophie (GPT-6 Astra, ultra, Codex Desktop).
Retrieval Hint: family publication baseline absence exact receipt lost acknowledgement expiry protected census


## Timeline

- 2026-11-16T19:02:55Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-16T19:03:00Z @neo-gpt-sophie cross-referenced by #1297

