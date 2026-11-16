---
id: 1509
title: Define the protected family protocol and temporal projection
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-06T23:06:00Z'
updatedAt: '2026-11-16T00:56:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1509'
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
blockedBy: []
blocking:
  - '[ ] 1510 Own protected family admission and coherent snapshots in GraphService'
closedAt: '2026-11-16T00:56:42Z'
---
# Define the protected family protocol and temporal projection

## Context and problem
The source checkpoint in #1314 mixes protocol, persistence and consumer changes. This leaf makes the pure contract independently reviewable without exposing a writer or changing runtime family decisions.

## Architectural reality and intended change
Reuse and finish the shared canonical JSON, operation/signature/key and vocabulary primitives in ai/services/shared/identityPublication*.mjs and the pure graph readers identityFoundingSeed.mjs / identityFamilyProjection.mjs. Prescription checked: these existing pure owners define bytes, keys and derived intervals; they do not own credentials, physical storage or admission authority. The checked-in roster is a reviewed setup input only; no new runtime identityRoots import.

## Contract Ledger
| Surface | Authority | Behavior and refusal | Evidence / docs |
| --- | --- | --- | --- |
| Canonical operation/envelope | C7 fixed profile | Exactly baseline/bind/retire/rekey/confirm/correct; canonical encodings, domain separation, fixed purpose/audience/issuer/kid and Ed25519 verification; 300000ms delivery validity | Existing canonical, key, operation and signature specs; IdentitySchema reference |
| Complete-baseline representation | Original #1296 AC-4 / agreed capacity | 128 managed bindings; derive the 588887-byte maximum including envelope/framing, without multipart/truncation | Worst-case encoded fixtures and named offending-entry/limit refusals |
| Family vocabulary and projection | C8, #700, founding/correction dispositions | Exact family labels, explicit model:null, complete membership diagnostics and corrected temporal intervals | Pure table/property tests and complete malformed-input controls |
| Publication canonicalizer and revision digest | Existing `ai/services/memory-core/helpers/publishDigest.mjs` exports and content-field projection | Move/re-export the deterministic canonicalizer through the shared owner. Preserve ordinary canonical bytes and publication digest normalization. The raw canonicalizer retains own `__proto__` keys; `computePublishRevisionDigest` still projects media onto its declared fields before canonicalization. | Direct raw-helper key-distinction control plus actual digest compatibility controls, including the projected-out extra-key case; retain scheduling and read-back behaviour. |

## Acceptance Criteria
- [ ] All six operations use one closed typed canonical profile. Reject duplicate/noncanonical encodings, wrong purpose/target/issuer/kid and invalid signature/time input; logical identity stays distinct from delivery identity.
- [ ] Stable provider-subject/incarnation keys survive display-login and same-incarnation re-key changes; remove/recreate is distinct. Complete baseline serialization honors the supported128-binding/588887-byte bound and refuses any unrepresentable entry or overflow as a whole.
- [ ] Preserve single-seat, different operators, same-owner/different-family, unconfirmed, empty, pending-retirement and retired-incarnation cases. Explicit model:null never inherits a previous model. Literal family "unknown" is distinct from missing/unclassified.
- [ ] Vocabulary comparison is NFKC → remove JavaScript whitespace → lower/upper/lower → NFKC; control/format characters and empty keys refuse. Preserve exact labels and safe own-key behavior. Include prototype-name and all-code-point case/idempotence controls. Cross-script confusables remain the recorded residual.
- [ ] The protected founding family continues through membership baseline and matching declarations. Cutoff is the earliest corrected different-family declaration at its ORIGINAL admission boundary, including retired incarnations. Test both correction directions, a later remaining cutoff, and no boundary at correction time. Membership/unconfirmed diagnostics remain visible; preserve the undeclared different-family residual and its cross-family-gate reopen trigger.
- [ ] Compile complete reviewed provider-bound founding input with a canonical digest: missing, duplicate, malformed or conflicting rows are not silently omitted; missing/malformed source never falls back to founding. Retain every founding row sharing a subject. Conflicting founding families remain ambiguous before the first corrected admitted declaration at its original admission time; from that boundary use the admitted aggregate, including unconfirmed/ambiguous states, without selecting or removing a founding row. Pure projection consumes complete detached input, never fetches two independent runtime snapshots.
- [ ] Focused primitive/projection suites and Brain preflight/CI pass, with no endpoint, writer, runtime identity-root fallback or consumer activation added.
- [ ] The existing publication helper re-exports the shared canonicalizer without changing its schedule/read-back APIs or content-field projection. Ordinary canonical bytes and publication digests stay equal; raw JSON objects differing only in an own `__proto__` key stay distinct after canonicalization. The actual digest still ignores unrecognized media fields, including that key, before canonicalization.

## Scope, mapping and dependencies
Original #1296: pure portions of AC-1,4,5,6,9,10 and the common AC-12 discipline. Physical enforcement remains with the storage leaf, real transport with the service leaf, and policy consumers with their leaves. [#112 was independently closed as superseded](https://github.com/neomjs/neo-agent-brain/issues/112#issuecomment-6903800861); it no longer retires the founding `properties.modelFamily` input. #1509/#1510 own that compiler input; any later representation change must preserve the complete seed digest. No installed census or authority is claimed by a pure compiler. Reject copying raw keys onto a plain `{}` because the prototype setter can erase an own key; retain the publication digest's explicit field projection.
Retrieval Hint: family canonical bytes typed operations founding cutoff corrected temporal projection

## Intake / independent review

[Grace’s independent epic review](https://github.com/neomjs/neo-agent-brain/issues/1296#issuecomment-6431284778) greenlights #1509–#1512. Its publication-compatibility concern is carried in the ledger and final AC above. A direct two-module probe against `c2b7859a` plus the shared checkpoint canonicalizer confirms the raw-helper key loss and correction, while the real `computePublishRevisionDigest` projects the extra key out both before and after the move. This is a raw canonicalizer correction with digest compatibility, not a new digest input field.

The [policy-owner disposition in review](https://github.com/neomjs/neo-agent-brain/pull/1774#pullrequestreview-5913484734) clarifies the conflicting-founders boundary in the final sentence of AC-6: ambiguity is historical until the first admitted declaration, not a permanent override of later admitted history. The single-family corrected cutoff and its undeclared-seat residual remain unchanged.

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

- 2026-11-06T23:06:02Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-06T23:06:02Z @neo-gpt-sophie added the `ai` label
- 2026-11-06T23:06:02Z @neo-gpt-sophie added the `architecture` label
- 2026-11-06T23:06:02Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-06T23:06:39Z @neo-gpt-sophie cross-referenced by #1510
- 2026-11-06T23:06:45Z @neo-gpt-sophie cross-referenced by #1511
- 2026-11-06T23:08:38Z @neo-gpt-sophie added parent issue #1296
- 2026-11-06T23:08:46Z @neo-gpt-sophie marked this issue as blocking #1510
- 2026-11-06T23:10:13Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-06T23:10:21Z @neo-gpt-sophie cross-referenced by #1297
- 2026-11-06T23:15:30Z @neo-gpt-sophie cross-referenced by PR #1314
### @neo-gpt-sophie - 2026-11-15T21:28:39Z

**Source packet for the independent review of #1296's five-leaf split.** This is an intake observation, not an implementation or test-pass claim.

At the preserved [`b6905782` checkpoint](https://github.com/neomjs/neo-agent-brain/tree/b69057823d43b813d3c0daa6421561a3d9fdc51a), the candidate extraction set named by this leaf is:

| Area | Source files | Existing matching specs |
| --- | --- | --- |
| Canonical values, identifiers, operations and envelope verification | `ai/services/shared/canonicalJson.mjs`, `identityPublicationKeys.mjs`, `identityPublicationOperation.mjs`, `identityPublicationSignature.mjs` | Four corresponding specs under `test/playwright/unit/ai/services/shared/` |
| Family vocabulary | `ai/services/memory-core/helpers/identityPublicationVocabulary.mjs` | Corresponding helper spec |
| Detached founding input and temporal projection | `ai/graph/identityFoundingSeed.mjs`, `identityFamilyProjection.mjs` | Two corresponding graph specs |

The seven source files total 1,118 lines at that checkpoint. Their existing specs total 1,367 lines. This is a size/import census only. The exact current `dev` tree at `c2b7859a` contains none of those named files; the draft is extraction evidence, not already delivered source.

The tests cannot be copied without reconciliation: the canonical-value spec also reads the existing `publishDigest` owner; the vocabulary spec reads `getCoreSwarmAgentFamilies`; the founding-input spec reads `FOUNDING_FAMILY_SUBJECTS`/`IDENTITIES` from the checkpoint's `identityRoots`. Those are explicit test seams to check against current `dev`, not permission to migrate runtime consumers or copy a roster into this leaf.

The client, protected admission/boot/fence/lineage helpers, storage/migration guards and their integration suites remain in #1510/#1511 and subsequent consumers. The two intentionally held migration-test edits in the resident checkout are untouched.

The native source sequence was re-read: **#1509 → #1510 → #1511 → #1512 → #1765**; #1513 has its separately recorded routing-relation gate. #1509 has no native blocker. Before first implementation pickup, the author path of `epic-review` requires a non-author review of #1296's current split; none of its 36 current comments has the structured epic-review form. Its historical peer dispositions remain authority inputs, not a claimed review of the later decomposition. No branch, assignment or tracked edit has been made.

- 2026-11-15T22:01:37Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-15T22:38:14Z @neo-gpt-sophie cross-referenced by PR #1774
- 2026-11-16T00:16:43Z @neo-fable-mnemosyne cross-referenced by #112
- 2026-11-16T00:22:11Z @neo-gpt-sophie referenced in commit `e872d49` - "fix(identity): bound founding precedence and validate lineage keys (#1509)"
- 2026-11-16T00:30:05Z @neo-gpt-sophie cross-referenced by #1513
- 2026-11-16T00:54:09Z @neo-gpt-sophie cross-referenced by #1512
- 2026-11-16T00:56:42Z @tobiu referenced in commit `1f884fe` - "feat(identity): add pure family protocol and temporal projection (#1509) (#1774)

* feat(identity): add pure family protocol and temporal projection (#1509)

* fix(identity): bound founding precedence and validate lineage keys (#1509)"
- 2026-11-16T00:56:43Z @tobiu closed this issue

