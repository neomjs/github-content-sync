---
id: 1511
title: Wire the protected family service through Memory Core
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-06T23:06:44Z'
updatedAt: '2026-11-16T17:00:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1511'
author: neo-gpt-sophie
commentsCount: 1
parentIssue: 1296
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1510 Own protected family admission and coherent snapshots in GraphService'
blocking:
  - '[ ] 1818 Read qualified family bootstrap and operation receipts'
  - '[ ] 1297 Publish admitted Fleet membership and manual family acts durably'
  - '[ ] 1513 Use admitted families for routing and publication checks'
  - '[ ] 1512 Use admitted family history for review decisions'
closedAt: '2026-11-16T17:00:59Z'
---
# Wire the protected family service through Memory Core

## Context and problem
At #1314's checkpoint, createIdentityPublicationClient and the admission helpers are exercised in fixtures, but the real MC publication/snapshot service is not registered. This leaf turns the completed source into a usable, authenticated service boundary.

## Architectural reality and intended change
Use shared AuthService and TransportService, MC Server/toolService/openapi/configBase, GraphService and the identityPublicationClient. MC trust is AiConfig-owned; resolved config is read at use sites. The client must be actually constructed by its production consumer composition, with proven caller identity and plane/root.

## Pickup and extraction findings — 16 November

Sophie owns this source lane. #1510 is closed through merged #1786; its native blocker edge remains as provenance. Aggregate draft #1314 is closed as superseded, with its b6905782 branch and original receipts retained. Intake is complete and the source candidate is [PR #1809](https://github.com/neomjs/neo-agent-brain/pull/1809). The current author-response head is `e1140422b97e4a61c7393547124e34e080f37ba7`; its optional-reader startup repair is tested locally, with fresh CI pending before re-review.

Current source at `4d3a5d1c68778770b6fefb2f481507007cb42772` provides GraphService's protected admission/snapshot methods, but the declared MC Server/toolService/OpenAPI boundary does not register the publication/snapshot composition yet.

The preserved client cannot be copied unchanged: `b6905782:ai/services/shared/identityPublicationClient.mjs` accepts a closed six-key snapshot and four-key count record, while the merged storage owner returns `founding`, `foundingRevision`, `foundingProvenance` and `counts.admittedSubjects` (`identityPublicationAdmission.mjs`, lines 485–489). Reconcile the client with the complete physical observation, including founding-only subjects; retain strict validation rather than dropping the new fields. The old literal founding-subject table is extraction evidence, not permission to provision current authority automatically.

## Contract Ledger
| Surface | Authority | Behavior and refusal | Evidence / docs |
| --- | --- | --- | --- |
| Class-9 publication POST | C7 and ADR0038/0041 | Dedicated typed route after common guards/before ordinary bearer chain; no generic PAT/MCP writer | Real HTTP route, purpose/auth/encoding negatives |
| Boot reservation, public trust and founding input | MC AiConfig + original-run lineage | Atomic `identityPublicationBoot` / `NEO_IDENTITY_PUBLICATION_BOOT` and optional reviewed `identityPublicationFounding` / `NEO_IDENTITY_PUBLICATION_FOUNDING`; pure profile/plane checks before any consumption, immutable seed install only after bound lineage | Actual MC boot, pending/destroyed startup, restart, exact seed retry and changed-seed refusal |
| Snapshot/lineage endpoints and client | Protected GraphService observation | `get_identity_publication_snapshot` returns one complete observation; `get_identity_publication_lineage` is read-only. Admission revision and founding digest/provenance remain distinct; caller and plane/root proofs gate initial/reconnected reads | Registered MCP operations + actual production client journey |
| Managed reader bootstrap | Existing plane credential owner and managed launch admission | Atomic `seat.planeTarget` / `NEO_SEAT_PLANE_TARGET` travels with the existing plane base/bearer into the local GitHub MCP row. Optional absence adds no ordinary GitHub requirement; retained selected-owner refusal stays a refusal. Recovery uses the original proven binding | Generated profile upgrade, Start serialization, redemption and clean recovery controls |

## Acceptance Criteria
- [ ] Register the actual dedicated publication handler and snapshot operation in Server/toolService/OpenAPI as appropriate, wired to GraphService's protected methods. The fixed application/vnd.neo.identity-family-publication carrier holds payload.signature; ordinary JSON parser limits remain unchanged.
- [ ] Through the real HTTP path, wrong purpose/audience/issuer/kid, duplicate/noncanonical encoding, expired/future delivery, ordinary seat/PAT/Bridge credentials and forged provenance refuse without mutation. Protected trust cannot be provisioned by an ordinary request.
- [ ] MC boot consumes the canonical reservation/public trust and exposes target-bound observations with stable originRunId/issuer/fingerprint; missing, unsupported, fenced or mismatched history refuses explicitly. No private key is generated here and no runtime config writes/pass-along defaults are introduced.
- [ ] The actual bounded transport admits a representable complete128-binding baseline and rejects a malformed/oversized588887-byte-contract violation without truncation or partial admission; reports entry/count/limit/remedy. Retain source-derived worst-case fixtures. The joint LIVE serialized census/admission witness is owned with #1297 under parent #1296, not a circular precondition on the publisher.
- [ ] Construct the identityPublicationClient in production using the declared credential owner. Bootstrap/reconnect prove authenticated identity AND served plane/root; no ambient bearer, local RequestContext shortcut or reader→Fleet pull. Changed identity/plane invalidates retained observations.
- [ ] The registered snapshot returns one coherent founding+admitted observation from the storage owner, including founding-only subjects and separate source provenance. Missing/malformed/fenced/unavailable never becomes empty success; do not fetch roster and history separately.
- [ ] Run the real handler/storage/client composition and interruption/refusal tests, config/OpenAPI parity and Brain preflight/CI. Source docs describe the exported contract and remedies. Publication is unavailable until the protected storage and trust prerequisites are satisfied; no temporary unguarded endpoint.

## Source implementation decisions — 16 November

The source uses a lifecycle-owned `IdentityPublicationService` and Neo client. A held graph cannot delay the WAL-local listener; publication/snapshot reads remain unavailable until trusted boot consumption succeeds, and destruction cancels pending authority work. Existing plane-coherence and AuthService checks run before the new consumption path. The original signed carrier reaches the physical admission transaction.

The production reader is created by the GitHub server after fresh viewer acquisition. Managed launch metadata forwards its independent target and optional plane credential; clean recovery derives public base/target only from the retained original credential binding. It never substitutes the repository PAT, host config, a newly selected tenant or returned family-snapshot fields.

The trusted founding input is a boot-only connection to #1510's installer, not a new request writer or automatic current-roster seed. #1297's provisioning composition must supply the reviewed input without adding a credential question. Existing first-install, immutable-content and original-provenance rules remain.

## Serving follow-ups

From [#1786’s review](https://github.com/neomjs/neo-agent-brain/pull/1786#pullrequestreview-5915272344): production composition pins the independently resolved plane/root and admission clock; request fields cannot select either. Exercise the registered path with forged target/clock inputs. Refusal remedies must distinguish verified retryable contention from missing, corrupt or fenced source; if the available diagnostic cannot establish that distinction, report it as unknown rather than promising that retry is safe. This does not change the signed operation grammar or permit empty success.

Before live serving, record Sophie’s recovery disposition under #1296 for a never-provisioned plane fenced by ordinary restore, coordinated with #1297’s provisioning/custody owner. Ordinary setup, retry and requests cannot clear that fence or recreate authority. This assigns the unresolved disposition; it does not claim a recovery mechanism or authorize one.

## Scope, mapping and dependencies
Blocked by #1510 (transitively #1509). Own actual original AC-1/3, wire/client portions of AC-4/9 and AC-12. This is the precise prerequisite for #1297; it does not wait for GitHub/routing consumer migration. No Fleet secret custody, outbox, UI, live service cut or D825 request-binding implementation.
Retrieval Hint: Memory Core class9 production handler snapshot client trusted boot identity plane binding

## Authority and delivery boundary
This is a one-PR source leaf of #1296, split from the preserved draft #1314 at b69057823d43b813d3c0daa6421561a3d9fdc51a by the operator's 6/7 November instruction. Extract the applicable work, then reconcile it against current dev; the old draft is evidence, not a complete implementation.

The accepted source is Institution [D811 C7/C8](https://github.com/neomjs/neo-agent-institution/discussions/811), Institution ADR 0038 C7/C8 and Brain ADR 0041. C7 body2026-10-26T04:29:27Z has GPT [AUTHOR_SIGNAL19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and Claude [GRADUATION_APPROVED19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284); C8 body2026-10-26T07:16:28Z has GPT [AUTHOR_SIGNAL19190332](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332) and Claude [approval19190177](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), extended to ACs A/B. No new policy is graduated by this split.

Decision Record impact: aligned with neomjs/neo-agent-brain#adr:aiconfig-reactive-provider-ssot and the protected-authority boundary; depends on the accepted ADR 0038/0041 records, whose tickets #1294 and neomjs/neo-agent-institution#818 are closed.
Unresolved dissent: none added to accepted C7/C8; the separate D825 request-binding delta remains unresolved and cannot be implemented by assumption.
Unresolved liveness: no absent peer is counted as consent. Sophie retains planning and source-custody stewardship and owns this source continuation. This protected-source continuation is not an unclaimed beginner task.

Every leaf must document named refusal remedies, run its actual production-path controls and current Brain preflight/CI, and retain failed experiments. Source evidence never substitutes for the installed manual Add → ownership → confirmation → genuinely required managed REQUEST_CHANGES witness, retained on neomjs/neo-agent-institution#414 under #700. No live seed, key generation, migration, deployment or service restart is authorized by these source tickets.

Structure map: ai:structure-map -- --files --loc passed in the resident Brain checkout. Existing owners are ai/graph, ai/services/shared, ai/services/memory-core and its helpers, ai/services/github-workflow, and ai/services/fleet; no new top-level subsystem. Any new helper follows its existing sibling pattern. ADR 0019's config read gate applies before config work.

Origin Session ID: 7367b697-abdc-4e46-963d-5c57378b3551
Authored by Sophie (GPT-6 Astra, ultra, Codex Desktop).

Creation sweep: live latest20 created-descending Brain issues, closed family/identity/publication decisions, own assigned #700/#1296–1299 and #112/#875 adjacency, current all-state A2A, KB and MC problem-noun recall checked immediately before filing on 2026-11-06 UTC (7 November local). No competing split found. #1296 is the existing source outcome; this leaf replaces part of its aggregate implementation, not the parent or sibling policy.


## Timeline

- 2026-11-06T23:06:44Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-06T23:06:45Z @neo-gpt-sophie added the `ai` label
- 2026-11-06T23:06:45Z @neo-gpt-sophie added the `architecture` label
- 2026-11-06T23:06:45Z @neo-gpt-sophie added the `security` label
- 2026-11-06T23:06:45Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-06T23:06:50Z @neo-gpt-sophie cross-referenced by #1512
- 2026-11-06T23:06:56Z @neo-gpt-sophie cross-referenced by #1513
- 2026-11-06T23:08:41Z @neo-gpt-sophie added parent issue #1296
- 2026-11-06T23:08:48Z @neo-gpt-sophie marked this issue as being blocked by #1510
- 2026-11-06T23:08:50Z @neo-gpt-sophie marked this issue as blocking #1512
- 2026-11-06T23:08:52Z @neo-gpt-sophie marked this issue as blocking #1513
- 2026-11-06T23:08:53Z @neo-gpt-sophie marked this issue as blocking #1297
- 2026-11-06T23:10:13Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-06T23:10:21Z @neo-gpt-sophie cross-referenced by #1297
- 2026-11-06T23:15:30Z @neo-gpt-sophie cross-referenced by PR #1314
- 2026-11-15T21:16:10Z @neo-gpt-sophie cross-referenced by #1765
- 2026-11-15T21:28:40Z @neo-gpt-sophie cross-referenced by #1509
- 2026-11-15T22:38:14Z @neo-gpt-sophie cross-referenced by PR #1774
- 2026-11-16T03:50:31Z @neo-gpt-sophie cross-referenced by PR #1786
- 2026-11-16T13:16:17Z @neo-gpt-sophie assigned to @neo-gpt-sophie
### @neo-gpt-sophie - 2026-11-16T13:29:02Z

**Intake — accepted with current-source reconciliation (16 November).** #1510 is closed and no competing #1511 implementation is open. The changed-surface drift probe fires, so the b690 checkpoint is not a cherry-pick prescription.

Prescription checked: `AuthService` owns the dedicated typed route after Transport's common guards; MC `Server` owns configured boot/trust composition; `GraphService` owns physical admission and coherent snapshots. These remain the correct owners. The consumer client must prove identity and remote plane/root at its actual production bootstrap, with no local-plane or returned-snapshot fallback.

ADR successor-risk: adr-aligned — Institution `adr:fm-client-topology` §2.5.3/2.5.4 and Brain `adr:bootstrap-record-verified-plane-handoff` §2.11 preserve the accepted C7/C8 boundary; `adr:aiconfig-reactive-provider-ssot` governs configuration. No new user credential question, authority policy, automatic provisioning or recovery path is introduced. The parent is self-authored; this work does not claim a new independent epic review.

Two concrete checkpoint changes are required: the current physical owner takes the original signed compact token, so the old AuthService callback cannot pass only its decoded envelope; and the client must validate the merged founding/history observation, including its additional fields and founding-only subjects. MC's existing WAL-local startup availability remains intact when graph publication is unavailable. Named negative tests will cover the real composed path; no live service operations are part of this source work.

The repository has no local `close-inactive-issues.yml`; no bot-band threshold is inferred. The live issue is open with no stale label, and current source/owner/dependency checks—not its age—establish readiness.

- 2026-11-16T14:59:26Z @neo-gpt-sophie cross-referenced by PR #1809
- 2026-11-16T15:19:29Z @neo-gpt-sophie referenced in commit `0270df5` - "fix(identity): correct family service catalog and CI fixtures (#1511)"
- 2026-11-16T16:38:43Z @neo-gpt-sophie referenced in commit `e114042` - "fix(github): keep optional family proof off the boot path (#1511)"
- 2026-11-16T17:00:59Z @tobiu referenced in commit `124deb9` - "feat(identity): serve protected family history through Memory Core (#1511) (#1809)

* feat(identity): serve protected family history through Memory Core (#1511)

* fix(identity): correct family service catalog and CI fixtures (#1511)

* fix(github): keep optional family proof off the boot path (#1511)"
- 2026-11-16T17:01:01Z @tobiu closed this issue
- 2026-11-16T18:25:44Z @neo-gpt-sophie cross-referenced by #1818
- 2026-11-16T18:37:58Z @neo-opus-grace cross-referenced by PR #1816

