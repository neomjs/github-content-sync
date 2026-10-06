---
id: 906
title: Carry FM repository trust into Claude Code
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-06T19:35:09Z'
updatedAt: '2026-10-06T19:36:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/906'
author: neo-gpt-emmy
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
# Carry FM repository trust into Claude Code

## Context
During the installed Candidate C migration on 2026-10-06, the operator selected the FM-managed Claude Code checkout and still had to accept the native workspace-trust dialog. This is an observed manual step, not proof that Claude exposes an automation API.

Design authority: @tobiu requested this feature and clarified its boundary: repositories already assigned to a peer inside FM should be trusted; arbitrary folders should not.

## The Problem
FM already records the operator's repository selection. Requiring the operator to rediscover that intent in each newly provisioned or relocated Claude workspace adds first-start friction. A folder merely residing beneath the agents root is not sufficient authority.

Anthropic documents per-directory workspace trust and separate MCP approval in its [security guide](https://code.claude.com/docs/en/security#additional-safeguards). Anthropic also documents the exact per-project grant, `projects["<path>"].hasTrustDialogAccepted = true` in `~/.claude.json`, in [workspace trust](https://code.claude.com/docs/en/permissions#what-runs-before-you-trust-a-folder). Native Desktop consumption of the implemented Fleet integration still needs an installed witness.

## The Architectural Reality
The Brain owns repository assignment and managed preparation:
- `ai/services/fleet/FleetManager.mjs:798`: `setRepos()` records assigned repositories.
- `ai/services/fleet/deriveAgentRepoPath.mjs:72`: derives the managed checkout path.
- `ai/services/fleet/startAgentProvisioned.mjs:76`: composes clone, preparation and launch; its repository contract also covers assigned secondary repositories.
- `ai/services/fleet/prepareManagedAgentWorkspace.mjs:1315`: already converges project-scoped Claude configuration with backup, atomic publication and concurrent-write detection, preserving resident trust fields today.
- `ai/services/fleet/inspectAgentRepo.mjs:17`: explicitly checks presence/checkout shape only; reused checkouts are not yet matched to the assigned remote. Trust must add that verification.

The structure-map command completed successfully; `ai/services/fleet` is the existing owning folder. No new service or source file is prescribed.

Closed #675 pins Claude auto memory and explicitly notes its dependence on trusted workspace settings. It does not implement this request.

## The Fix
Extend the existing Claude preparation boundary to honor an operator's current FM repository assignment for its verified checkout, using the documented per-project trust key. Verify consumption against the installed Claude Code surface and intended configuration recipient.

Match the resolved checkout origin to the assigned forge/repository before applying trust; directory name or `.git` presence alone is insufficient. Resolve Claude's canonical trust root. For worktrees it is the main checkout root, so refuse a grant that would extend beyond the assigned repository's verified scope. Preserve explicit newer distrust and handle relocation as verification of the same assigned repository at its new destination. Repeated preparation must be idempotent.

If the vendor provides no supported mechanism, publish that evidence and leave the automation outcome explicitly blocked; do not mark it implemented using a permission bypass or manual-only workaround.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Assigned Claude checkout at preparation/launch | Current FM repository assignment via `FleetManager.setRepos()`, plus verified checkout identity | Native workspace trust applies to that checkout only | Existing native prompt when integration is unsupported or verification fails; never report automatic success | Update existing Fleet preparation guidance with the verified support boundary | Native first-start witness and negative controls |
| Relocated assigned checkout | Same assignment and verified destination | Trust follows the verified repository destination without trusting the parent root | Preserve native prompt and report unmet automation | Same guidance | Relocation/repeat witness; unrelated folder remains untrusted |

## Decision Record impact
Aligned with the existing preparation boundary: carries the existing operator assignment into a harness integration. If implementation requires changing trust authority or a cross-harness policy, return to architectural review before expanding this scope.

## Acceptance Criteria
- [ ] Use the documented project trust key; record supported Claude versions/surfaces and prove it addresses the intended configuration recipient.
- [ ] An assigned, verified repository opens trusted on first start and after a verified root relocation, without a second workspace-trust click.
- [ ] Repeated preparation is idempotent and preserves explicit newer distrust.
- [ ] Unassigned folders, sibling repositories, checkout identity mismatches and escaping symlink targets receive no trust from this integration.
- [ ] Do not rewrite permission rules/modes, hook approvals or MCP approval records. Document that native trust enables the repository's existing trust-gated allow rules and additional directories.
- [ ] Record an installed Claude Code witness for the positive and negative cases. Fixture-only success is not native acceptance.

## Out of Scope
Model/effort selection, selecting the Code folder, dependency installation, arbitrary-path trust, changing the seat's repository selection, and general permission approval.

## Avoided Traps
No blanket agents-root trust, hidden approval-database patching, `bypassPermissions`, or extra generic trust switch. FM's existing repository assignment supplies the intent.

## Related and creation evidence
Parent: #571. Installed acceptance: neomjs/neo-agent-institution#12.
Live latest-open sweep: latest 20 Brain issues checked at 2026-10-06 19:34 UTC; no equivalent. Recent all-status A2A slice (30 messages) has no competing claim. Memory Core rationale sweep recovered repository-assignment ownership, not a native trust solution; own open assignments have no overlap. Historical search found the distinct completed #675.

Retrieval Hint: "FM assigned repositories Claude workspace trust Candidate C"
Origin Session ID: d0d0bed3-7ce4-4bce-a16d-59589484aec0



## Timeline

- 2026-10-06T19:35:09Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-06T19:35:11Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-06T19:35:11Z @neo-gpt-emmy added the `ai` label
- 2026-10-06T19:35:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-06T19:35:32Z @neo-gpt-emmy added parent issue #571
- 2026-10-06T19:37:34Z @neo-gpt-emmy cross-referenced by #12

