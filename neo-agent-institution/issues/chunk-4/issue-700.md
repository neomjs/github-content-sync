---
id: 700
title: Remove development-only integration notes from the public README
state: CLOSED
labels:
  - bug
  - documentation
  - ai
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-15T15:27:07Z'
updatedAt: '2026-10-15T16:20:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/700'
author: neo-gpt-euclid
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
closedAt: '2026-10-15T16:20:32Z'
---
# Remove development-only integration notes from the public README

## Context

The operator found development-host integration instructions in the public README and requested their removal. The public entry point should describe available Fleet Manager functionality without requiring access to unpublished integration material.

## The Problem

A README section currently includes an inaccessible repository link and a separate-checkout development command. Publishing implementation details there creates an unsupported product expectation. This correction should not repeat the removed integration details or make availability promises.

## The Architectural Reality

`README.md` is the public user-facing entry point. The development-host capability is independently opt-in; documentation removal does not require a runtime or dependency change. Integration-specific operational material belongs with its owning development project.

## The Fix

Delete the complete development-sibling section. Audit tracked handwritten Markdown for equivalent links or setup guidance. Preserve existing runtime omission/default behaviour; do not add a replacement public advertisement or generic extension API.

Prescription checked: `README.md` owns the incorrect public guidance; existing capability gating owns optionality and is outside this documentation patch.

## Acceptance Criteria

- [ ] AC-1: The public README no longer includes the development integration section, inaccessible repository link or separate-checkout command.
- [ ] AC-2: The nearby tracked Markdown audit records any equivalent references; the resulting change is documentation-only and leaves the capability opt-in.

## Out of Scope

Runtime changes, dependencies, installation, release promises, extension architecture and history rewriting.

## Avoided Traps

Do not replace the removed section with a teaser for an unpublished addition. Do not repeat the removed integration details in the new public ticket or PR prose.

Decision Record impact: none.

## Related

Operator-requested documentation correction to the current README.

Live latest-open sweep: latest 20 open issues and scoped README/integration search checked on 2026-10-15; no equivalent. Latest 30 all-state A2A subjects show no competing documentation claim. Own assignments cover security and installed-state evidence, not this correction. MC symptom searches found no contrary publication decision; the current operator instruction is the authority.

Origin Session ID: ed9c23e6-01d5-4dc2-8031-cb7c5a631f62

Retrieval Hint: public README development-only integration guidance.


## Timeline

- 2026-10-15T15:27:07Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-15T15:27:08Z @neo-gpt-euclid added the `bug` label
- 2026-10-15T15:27:08Z @neo-gpt-euclid added the `documentation` label
- 2026-10-15T15:27:09Z @neo-gpt-euclid added the `ai` label
- 2026-10-15T15:30:17Z @neo-gpt-euclid cross-referenced by PR #701
- 2026-10-15T16:20:33Z @tobiu referenced in commit `bc65a19` - "docs(readme): remove development-only integration guidance (#700) (#701)"
- 2026-10-15T16:20:33Z @tobiu closed this issue
- 2026-10-17T20:35:00Z @neo-fable-clio cross-referenced by PR #729
- 2026-10-26T05:40:45Z @neo-gpt-sophie cross-referenced by PR #817
- 2026-10-26T08:31:10Z @neo-gpt-sophie cross-referenced by #819

