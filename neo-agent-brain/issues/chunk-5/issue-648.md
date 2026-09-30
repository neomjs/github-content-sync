---
id: 648
title: Fleet rejects Codex restarts after native home settings change
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T19:23:05Z'
updatedAt: '2026-09-30T19:44:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/648'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-30T19:44:06Z'
---
# Fleet rejects Codex restarts after native home settings change

## Context
A real Fleet-managed Codex Desktop restart on 2026-09-30 failed after the resident's first session. The installed Brain revision was `ba470d8`; the log reported `FLEET_WORKSPACE_DIVERGENT`, owned key `projects.<managed-repo>.trust_level`, reason `Fleet trust block diverged`. Independent TOML parsing still reads the managed repository's trust as `trusted`.

## The Problem
Codex had placed ordinary root settings (`notify`, `model`, `model_reasoning_effort`) after Fleet's begin comment and before the project table. These settings were present in the stopped-profile backup. The comment boundary is not an ownership boundary for native Codex's writer. A byte-for-byte comparison against the originally rendered block rejects unchanged trust and prevents another launch.

## The Architectural Reality
`ai/services/fleet/prepareManagedAgentWorkspace.mjs:1417` owns `convergeCodexRemoteTrust`; its remote re-entry arm compares the entire marked block. The same file already imports `smol-toml` for native configuration semantics. Both `codex` and `codex-desktop` use this home-policy convergence. Fleet should evaluate its trust key while preserving resident-authored bytes.
Structure map: `npm run ai:structure-map -- --files --loc` passed; this remains in the existing Fleet service and canonical sibling spec.

## The Fix
Validate the requested repository's actual TOML trust value on remote re-entry. Preserve all home configuration and authentication bytes when it is already trusted. Keep malformed-marker, invalid-TOML, missing/wrong-target trust, and explicit trust-downgrade refusals bounded and non-destructive. Do not broaden the opt-out removal path: a mixed marked block must not delete resident settings.

## Contract Ledger
| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `convergeCodexRemoteTrust` remote re-entry | Existing helper's trust-only intent and observed native home layout | Accept semantic `projects[repoPath].trust_level === "trusted"` despite unrelated settings within markers; preserve bytes | Refuse missing/wrong-target/non-trusted trust or malformed TOML/markers without echoing source | Helper JSDoc | Both Codex harnesses: first prepare, native-shaped settings insertion, second prepare |
| Remote-to-stdio removal | Existing marked-block removal ownership | Retain exact-owned-block removal | Mixed resident/Fleet block remains a refusal; no resident deletion | Helper JSDoc | Regression for mixed-block opt-out and existing transport suite |

## Decision Record impact
None. This restores narrow existing configuration ownership; no new trust grant, transport, configuration authority, or service boundary.

## Acceptance Criteria
- [ ] Both Codex harnesses re-enter successfully after root model/notify settings appear inside Fleet comments, with home and authentication bytes unchanged.
- [ ] Equivalent valid TOML trust spelling is accepted; invalid TOML, missing/wrong-target trust, an explicit downgrade, and malformed markers fail without exposing source values.
- [ ] Mixed-block opt-out cannot remove resident settings; exact-block opt-out still works.
- [ ] Existing workspace preparation tests pass; a native-layout reproduction fails before the change and passes after it.
## Post-Merge Validation
- [ ] **[L4-deferred — operator handoff needed; owner: neomjs/neo-agent-institution#12]** Install the reviewed, merged repair in the macOS app and retry the stopped resident through Fleet. Record launch and effective context separately from configuration convergence.

## Out of Scope
Model capacity policy, wake subscriptions, repository instruction generation, provider accounts, and global trust changes.

## Avoided Traps
Rewriting the resident's home into Fleet's old layout would hide a repeatable product bug. Removing an entire mixed marked block could destroy native settings. Neither is needed for remote re-entry.

## Adjacency sweep
Latest 20 open Brain issues and 30 all-status A2A rows checked immediately before filing on 2026-09-30: no equivalent trust re-entry claim. Own assignments `#426`, `#306`, `#48` do not cover this defect. Problem-shaped Memory Core searches found the onboarding lineage but no prior repair; KB did not resolve this helper. Grace's `#644` touches instruction projection in the same file, outside these trust helpers.

## Related
- PR #643 (larger-context seed rollout exposed this restart refusal)
- #644 (separate instruction projection)
- neomjs/neo-agent-institution#12 (installed onboarding evidence)

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: `Codex Fleet trust block diverged notify model restart`

— Emmy, GPT-6 Astra Ultra, Codex.


## Timeline

- 2026-09-30T19:23:06Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T19:23:08Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T19:23:08Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T19:23:08Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T19:29:31Z @neo-opus-vega cross-referenced by #79
- 2026-09-30T19:30:32Z @neo-gpt-emmy cross-referenced by PR #649
- 2026-09-30T19:44:06Z @tobiu referenced in commit `408ac57` - "Merge pull request #649 from neomjs/codex/648-codex-trust-reentry

fix(fleet): preserve native Codex settings on restart (#648)"
- 2026-09-30T19:44:06Z @tobiu closed this issue

