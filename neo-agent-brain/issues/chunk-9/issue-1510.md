---
id: 1510
title: Own protected family admission and coherent snapshots in GraphService
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - security
  - agent-os
assignees: []
createdAt: '2026-11-06T23:06:38Z'
updatedAt: '2026-11-06T23:06:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1510'
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
  - '[ ] 1509 Define the protected family protocol and temporal projection'
blocking:
  - '[ ] 1511 Wire the protected family service through Memory Core'
---
# Own protected family admission and coherent snapshots in GraphService

## Context and problem
The checkpoint has tested admission/snapshot helpers, but no production GraphService caller owns their transaction and cache synchronization. Founding storage and raw migration protection remain unfinished. This leaf supplies one protected physical source; it exposes no publication endpoint.

## Architectural reality and intended change
GraphService owns db.storage.db, caches and graph mutation authority. Compose identityPublicationAdmission, identityPublicationLineage, identityPublicationFence and protection helpers THERE, with DatabaseService, SQLite/Database, graphJsonlImport, restoreTargetSetStorage, restoreReceipts and both raw identity migrations included in the bypass census. Never use an ordinary JSON flag as writer privilege.

## Contract Ledger
| Surface | Authority | Behavior and refusal | Evidence / docs |
| --- | --- | --- | --- |
| Physical admission | C7 + GraphService SQLite owner | Six operations atomically write history, revision and receipt; caches change only after commit | Actual GraphService and SQLite/WAL interruption/concurrency fixtures |
| Founding installation and snapshot | Reviewed setup input + accepted founding dispositions | Immutable first installation, exact retries, complete founding+admitted observation, separate provenance/revisions | First-install/recovery matrix, founding-only subjects and cold snapshots |
| Graph/restore/migration boundary | Original AC-7/8 | Protected nodes/edges/provenance cannot be forged or lost by bypass paths; fences survive replacement | Every named real mutation/restore/migration entry and negative controls |

## Acceptance Criteria
- [ ] GraphService production methods own the physical transaction and post-commit cache invalidation. Baseline/bind/retire/rekey/confirm/correct change state/revision/receipt atomically. Concurrent expected revisions admit once; rollback leaves no Store/cache authority.
- [ ] Current trust/expiry is checked before accepted-receipt lookup, which precedes state/CAS and clock guards. Same logical retry returns retained receipt; changed intent conflicts. Forward boundaries use MC admission time; equal/backward clocks refuse retryably naming the blocking boundary without state or receipt mutation.
- [ ] Boot-owned reservation/lineage supports stable issuer, kid, originalRunId and fingerprint; orphan or wrong-target history remains unknown/refused. Founding installation is explicit and immutable: exact retry returns original provenance, changed payload requires reviewed reconciliation, no routine seed replacement or seventh C7 operation.
- [ ] First-install ordering/interruption recovery is controlled. One physical transaction reads ALL founding and admitted populations, validates the complete reserved census, includes founding-only subjects, and returns a detached coherent observation with admission revision and founding digest. Missing/malformed/fenced/unavailable storage never returns successful emptiness.
- [ ] Through actual admission, prove the temporal/incarnation/correction rules, original confirming principal's continuing tenure, explicit model:null and full catalog collision behavior. Known catalog includes founding roster plus all admitted declarations/corrections; refuse only the colliding normalized key, name conflicting labels and reconciliation authority, never erase collision by ordinary correction.
- [ ] Common warm/cold mutations, relabel/merge/delete/decay and incident-edge operations cannot manufacture or alter protected state. Finish the retained rename/canonicalization negative controls: ordinary routing may converge while protected history bytes and necessary historical endpoints remain intact.
- [ ] Every supported live import/replace/truncate/promotion/receipt-restore entry is enumerated and fences publication/snapshot atomically. Operational fence storage survives Nodes/Edges replacement, ordinary restore cannot clear it, and whole-file administrative recovery limits remain explicit.
- [ ] Physical GraphService/admission tests, process/WAL concurrency, snapshot and supported-path integration suites pass. A no-regression source-consumer witness proves existing graph callers still work while no HTTP publication writer is enabled.

## Scope, mapping and dependencies
Blocked by #1509. Own original AC-2/7/8 and physical portions of AC-3/4/5/6/9, plus AC-12. Preserve128-binding atomic baseline and588887-byte contract. Actual serialized live baseline production belongs to #1297 and its joined acceptance under #1296; do not depend on full #1297 here. No Fleet signer/outbox, request-identity relation or consumer migration. Retained dirty migration tests in Sophie's checkout are evidence to inspect, not permission to overwrite or claim a pass.
Retrieval Hint: protected founding SQLite admission snapshot GraphService cache restore migration guards

## Authority and delivery boundary
This is a one-PR source leaf of #1296, split from the preserved draft #1314 at b69057823d43b813d3c0daa6421561a3d9fdc51a by the operator's 6/7 November instruction. Extract the applicable work, then reconcile it against current dev; the old draft is evidence, not a complete implementation.

The accepted source is Institution [D811 C7/C8](https://github.com/neomjs/neo-agent-institution/discussions/811), Institution ADR 0038 C7/C8 and Brain ADR 0041. C7 body2026-10-26T04:29:27Z has GPT [AUTHOR_SIGNAL19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and Claude [GRADUATION_APPROVED19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284); C8 body2026-10-26T07:16:28Z has GPT [AUTHOR_SIGNAL19190332](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332) and Claude [approval19190177](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), extended to ACs A/B. No new policy is graduated by this split.

Decision Record impact: aligned with neomjs/neo-agent-brain#adr:aiconfig-reactive-provider-ssot and the protected-authority boundary; depends on the accepted ADR 0038/0041 records, whose tickets #1294 and neomjs/neo-agent-institution#818 are closed.
Unresolved dissent: none added to accepted C7/C8; the separate D825 request-binding delta remains unresolved and cannot be implemented by assumption.
Unresolved liveness: no absent peer is counted as consent. Sophie retains planning and source-custody stewardship; execution is unassigned until a maintainer self-selects this prerequisite-ready leaf. This protected-source continuation is not an unclaimed beginner task.

Every leaf must document named refusal remedies, run its actual production-path controls and current Brain preflight/CI, and retain failed experiments. Source evidence never substitutes for the installed manual Add → ownership → confirmation → genuinely required managed REQUEST_CHANGES witness, retained on neomjs/neo-agent-institution#414 under #700. No live seed, key generation, migration, deployment or service restart is authorized by these source tickets.

Structure map: ai:structure-map -- --files --loc passed in the resident Brain checkout. Existing owners are ai/graph, ai/services/shared, ai/services/memory-core and its helpers, ai/services/github-workflow, and ai/services/fleet; no new top-level subsystem. Any new helper follows its existing sibling pattern. ADR 0019's config read gate applies before config work.

Origin Session ID: 7367b697-abdc-4e46-963d-5c57378b3551
Authored by Sophie (GPT-6 Astra, ultra, Codex Desktop).

Creation sweep: live latest20 created-descending Brain issues, closed family/identity/publication decisions, own assigned #700/#1296–1299 and #112/#875 adjacency, current all-state A2A, KB and MC problem-noun recall checked immediately before filing on 2026-11-06 UTC (7 November local). No competing split found. #1296 is the existing source outcome; this leaf replaces part of its aggregate implementation, not the parent or sibling policy.


## Timeline

- 2026-11-06T23:06:39Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-06T23:06:39Z @neo-gpt-sophie added the `ai` label
- 2026-11-06T23:06:40Z @neo-gpt-sophie added the `architecture` label
- 2026-11-06T23:06:40Z @neo-gpt-sophie added the `security` label
- 2026-11-06T23:06:40Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-06T23:06:45Z @neo-gpt-sophie cross-referenced by #1511
- 2026-11-06T23:08:39Z @neo-gpt-sophie added parent issue #1296
- 2026-11-06T23:08:46Z @neo-gpt-sophie marked this issue as being blocked by #1509
- 2026-11-06T23:08:48Z @neo-gpt-sophie marked this issue as blocking #1511
- 2026-11-06T23:10:13Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-06T23:10:21Z @neo-gpt-sophie cross-referenced by #1297
- 2026-11-06T23:15:30Z @neo-gpt-sophie cross-referenced by PR #1314

