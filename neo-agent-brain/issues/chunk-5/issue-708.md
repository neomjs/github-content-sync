---
id: 708
title: Align Memory Core auth reference with credential-based identity
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-01T17:29:14Z'
updatedAt: '2026-10-01T17:57:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/708'
author: neo-gpt-sophie
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
closedAt: '2026-10-01T17:57:40Z'
---
# Align Memory Core auth reference with credential-based identity

## Context

The follow-up to #694 / #695 is the existing authentication reference, `learn/agentos/tooling/MemoryCoreMcpAuth.md`. Its onboarding and troubleshooting instructions still turn Neo's optional team roster into a general prerequisite. The discrepancy was independently checked against Brain dev `110be14b19bba37ae72c7ee04756bcd431f85738`.

## The Problem

The transport table and request-time provisioning explanation cover OIDC and GitLab PAT but omit GitHub PAT and the other current modes. The unbound-identity recipe tells any new account to edit `identityRoots.mjs` and seed/restart. A reader following that advice for a credential-backed HTTP deployment gets the wrong admission model. The current `OwnAgentTeam.md` already explains the corrected boundary.

## The Architectural Reality

- `ai/configBase.mjs:624-625,734-746,2676-2677`: `auth.mode` selects five modes. Without an override, `auth.autoProvisionIdentitySources` is the singleton active PAT mode for `github-pat` or `gitlab-pat`, and empty otherwise; an explicit empty override disables provisioning.
- `ai/mcp/server/memory-core/Server.mjs:495-519`: no `userId` gives an empty request context. An allowed auth source takes the provisioning path; another identity-bearing source binds an existing node.
- `Server.mjs:707-719,739-779`: stdio resolves identity at boot and looks up the existing graph node; missing binding is distinct from failed authentication.
- `ai/mcp/server/shared/services/AuthService.mjs` owns credential validation. Local bearer is possession-only; seat tokens carry their minted subject.
- `learn/agentos/OwnAgentTeam.md` is the adjacent corrected onboarding explanation. The optional Neo roster is team metadata, not the general credential admission authority.

## The Fix

Correct the existing reference in place: transport/mode coverage, provisioning defaults and overrides, graph binding, context shape, and mode-specific troubleshooting. Scope `NEO_AGENT_IDENTITY` examples to local stdio. Preserve valid stdio lookup diagnostics and optional roster seeding for teams that actually use a roster. Align the existing diagram and OAuth-specific instructions with those boundaries.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Authentication and binding instructions | `AuthService.mjs`; `Server.mjs#buildRequestContext`, `#resolveStdioIdentity`, `#bindAgentIdentity`; `configBase.mjs` auth formula | Describe the selected mode, configured provisioning policy, existing-node lookup, and optional roster accurately | Missing binding diagnoses the active path; no universal roster or auto-create promise | `learn/agentos/tooling/MemoryCoreMcpAuth.md` | AC-1–3 |

Decision Record impact: none — document implemented behavior; no runtime or authority change.

## Acceptance Criteria

- [ ] AC-1 The transport/mode and graph-binding sections reflect all five modes and the active-PAT default, explicit override, and lookup-only cases.
- [ ] AC-2 The harness examples and unbound diagnostics distinguish stdio from HTTP; roster seeding is scoped to teams using that optional roster.
- [ ] AC-3 Context examples, OAuth instructions, and the existing diagram do not imply every authenticated request has an identity or provisions a node. Links and source anchors are checked.
- [ ] AC-4 The PR records exact source grounding and documentation validation; no runtime/credential changes.

## Out of Scope

Changing authentication, graph provisioning, roster metadata, credentials, deployment configuration, or the separate family-metadata work in #700. No new instruction substrate or new guide.

## Avoided Traps

Do not replace the old roster prerequisite with a universal first-request auto-provision promise: unlisted sources bind existing nodes, and local bearer supplies no identity. Keep this tooling reference focused; the broader onboarding narrative belongs in `OwnAgentTeam.md`.

## Related

#694, #695, #665

## Sweeps

Live latest-open sweep: latest 20 created-descending open Brain issues at 2026-10-01 17:28Z, with author/labels/URL; no equivalent. Exact all-state `MemoryCoreMcpAuth` search returned #37 (KB retrieval validation) and closed #10 (guide migration), neither this correction. A2A latest 30 all-status messages through 17:27Z: no competing claim. MC problem-noun sweeps returned the earlier `#695` provisioning correction and unrelated history; no ruling against this follow-up. KB ticket sweep surfaced the original PAT provisioning and local parity work, not a duplicate. Own-assignment sweep: zero open Brain issues.

Structure map: `npm run ai:structure-map -- --files --loc` is not hosted in this Engine checkout (Missing script); the owning surface is the existing Brain tooling reference beside the corrected `OwnAgentTeam.md`. No new file placement.

Origin Session ID: c364ebda-af03-4392-ae57-3d129e60b1df
Retrieval Hint: "MemoryCoreMcpAuth identityRoots optional roster active PAT provisioning"


## Timeline

- 2026-10-01T17:29:14Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-01T17:29:16Z @neo-gpt-sophie added the `bug` label
- 2026-10-01T17:29:16Z @neo-gpt-sophie added the `documentation` label
- 2026-10-01T17:29:16Z @neo-gpt-sophie added the `ai` label
- 2026-10-01T17:40:14Z @neo-gpt-sophie referenced in commit `a28dd0a` - "docs(memory-core): align auth reference with credential identity (#708)"
- 2026-10-01T17:40:57Z @neo-gpt-sophie cross-referenced by PR #709
- 2026-10-01T17:57:40Z @tobiu referenced in commit `2e46930` - "docs(memory-core): align auth reference with credential identity (#708) (#709)"
- 2026-10-01T17:57:40Z @tobiu closed this issue

